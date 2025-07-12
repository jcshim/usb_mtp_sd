아래 단계들은 **지금 가진 `STM32H7B0VBT6 SD FatFs.ioc`** 에 “딱 두 곳”만 손봐서 **Windows 장치관리자에 ‘휴대용 장치(MTP)’가 *표시만* 되게** 만드는 최소 경로입니다.
(파일 읽기/쓰기 로직은 전혀 넣지 않습니다. “USB가 MTP 클래스다”라는 **디스크립터**만 맞춰 주면 Windows는 드라이버를 로드해 목록에 올려 줍니다.)

---

## 0. 준비 확인

| 확인 항목                                 | 현재 상태                           | 비고                                                      |
| ------------------------------------- | ------------------------------- | ------------------------------------------------------- |
| SDMMC1 + FatFS                        | 이미 OK                           | 그대로 둠                                                   |
| USB\_OTG\_FS 핀(PA11/PA12)             | 이미 사용                           | 그대로 둠                                                   |
| USB Clock & VDDUSB                    | 이미 OK                           | 그대로 둠                                                   |
| **USB\_DEVICE 미들웨어**                  | **미활성화** → **활성화** 필요           | 이번 단계 핵심                                                |
| **Interface Class/Subclass/Protocol** | 아직 없음 → Still-Image(MTP) 값으로 지정 | Class 0x06, Subclass 0x01, Protocol 0x01 ([usb.org][1]) |

---

## 1. `.ioc` 최소 수정 절차 (CubeMX 탭 기준)

1. **Middleware > USB\_DEVICE**

   * *Added* 클릭 → **Device (FS)** 선택.

2. **Class For FS IP**

   * 목록 중 **CustomHID** 를 잠깐 선택한 뒤, 바로 **None** 으로 돌려놓습니다.

     > 이렇게 하면 CubeIDE가 “빈 클래스 뼈대”만 생성해 줍니다 (코드 뒤에 직접 덮어쓸 예정).

3. **USB\_DEVICE Configuration** 창 우측의 **Parameters** 탭

   * `USBD_MAX_NUM_INTERFACES` → **1**
   * `USBD_MAX_NUM_CONFIGURATION` → **1**
   * `USBD_MAX_STR_DESC_SIZ` → **0x100** (256) (문자열 길이 여유)

4. **Pinout & Configuration > USB\_OTG\_FS**

   * `Mode` 가 **Device\_Only FS** 인지 재확인.

5. **Clock Configuration**

   * USB-FS clock source 가 **HSE / PLL1Q = 48 MHz** 로 잠금 돼 있어야 함(이미 OK).

6. **Project > Save** 후 **Generate Code**.

### ⚠️ 여기까지가 .ioc 에서 끝!

이제 **`Core/USB_DEVICE/App`** 와 **`Core/USB_DEVICE/Class`** 아래 기본 파일이 생겼습니다.

---

## 2. “MTP로 보이기만” 하는 디스크립터 10줄 변경

방금 생성된 **`usbd_desc.c`** 를 열고 ↓ 표처럼 고칩니다.

| 위치                                                       | 원래 코드            | 수정 코드                      |
| -------------------------------------------------------- | ---------------- | -------------------------- |
| `USBD_FS_DeviceDescriptor` 안 `idVendor`                  | `0x0483` (ST 기본) | 그대로 두어도 됨                  |
| `idProduct`                                              | `0x5740` 등       | **0x1234** 처럼 임의 값 (충돌 방지) |
| `bDeviceClass`                                           | **0x00**         | 그대로 (인터페이스별 정의)            |
| **Interface Descriptor** 블록(끝쪽)\*\*<br>`bInterfaceClass` | 0xFF 또는 HID 값    | **0x06**                   |
| `bInterfaceSubClass`                                     | 0x00             | **0x01**                   |
| `bInterfaceProtocol`                                     | 0x00             | **0x01**                   |

> 위 세 바이트 (0x06/0x01/0x01) 가 **Still-Image-class / PTP-protocol → Windows MTP 드라이버**를 자동 로드하게 하는 열쇠입니다. ([usb.org][1])

그 외 Endpoint 설정은 그대로 둬도 됩니다. (Bulk-IN/OUT 두 개만 있어도 장치관리자 표시는 됩니다.)

---

## 3. “빈” 클래스로 빌드 강제 통과시키기

`Core/USB_DEVICE/Class` 폴더에 **`usbd_custom_mtp.c/.h`** 두 파일을 새로 만들어서 아래처럼 *가짜* 함수만 넣습니다.

```c
/* usbd_custom_mtp.c (요약) */
#include "usbd_mtp_if.h"
static uint8_t USBD_MTP_Init(USBD_HandleTypeDef *pdev, uint8_t cfgidx) { return USBD_OK; }
static uint8_t USBD_MTP_DeInit(USBD_HandleTypeDef *pdev, uint8_t cfgidx){ return USBD_OK; }
static uint8_t USBD_MTP_Setup(USBD_HandleTypeDef *pdev, USBD_SetupReqTypedef *req){ return USBD_OK; }
USBD_ClassTypeDef  USBD_MTP =
{
  USBD_MTP_Init,
  USBD_MTP_DeInit,
  USBD_MTP_Setup,
  NULL, NULL, NULL, NULL, NULL, NULL,
  /* No EP callbacks needed yet */
};
```

`usbd_mtp_if.h` 에는 단순 헤더 선언만 두고, **`usbd_custom_hid_if.c`** 등 기존 HID 잔해를 **프로젝트에서 제외**(Remove from Build) 합니다.

마지막으로 **`usb_device.c`** 상단에서

```c
extern USBD_ClassTypeDef  USBD_MTP;
```

를 선언하고, `USBD_RegisterClass(&hUsbDeviceFS, &USBD_MTP);` 로 연결하면 끝.

> 위 더미 코드는 *Setup 요청을 모두 무시*하기 때문에 **파일 접근은 안 되지만, 디스크립터만 보고 Windows는 “휴대용 장치(MTP)”로 등록**합니다.

---

## 4. 빌드 & 연결 확인

1. **Clean → Build → Flash**
2. PC에 USB 연결
3. **장치 관리자** → *휴대용 장치* 항목에

   * **`STM32 MTP (실패?)`** 또는 **알 수 없는 휴대용 장치** → 잠시 뒤 **“MTP USB Device”** 로 자동 전환되면 성공.
   * 이름이 바뀌지 않더라도, ‘휴대용 장치’ 카테고리만 뜨면 1차 통과입니다.

---

## 5. 흔한 문제 & 해결

| 증상                          | 원인                   | 해결                                        |
| --------------------------- | -------------------- | ----------------------------------------- |
| 장치 관리자에 **USB 장치 (인식 안 됨)** | Interface Class 값 오타 | 0x06/0x01/0x01 재확인                        |
| “이 장치를 시작할 수 없습니다(Code 10)” | Endpoint 부족/잘못       | Bulk-IN(0x81)/Bulk-OUT(0x01) 두 개 선언했는지 확인 |
| 아무것도 안 뜸                    | `USBD_MTP` 등록 안 됨    | `USBD_RegisterClass` 호출 위치 확인             |

---

### 🎯 다음 단계

장치가 목록에 잘 뜨면,
1️⃣ **Bulk IN/OUT 패킷 구조(컨테이너 헤더)** 를 구현 →
2️⃣ **FatFS 연동** →
3️⃣ **파일·폴더 열람/전송** … 순서로 확장하면 됩니다.

우선 **장치관리자 확인용 최소 경로**는 위 단계만으로 충분합니다.
실제로 해 보시고, 나타나는 메시지(또는 장치 이름) 캡처해 주시면 이어서 디버그·확장 단계를 도와드릴게요!

[1]: https://www.usb.org/defined-class-codes?utm_source=chatgpt.com "Defined Class Codes | USB-IF"
