## 프람프트
```
stm32h7로, MicroSD 카드를 위한 MTP를 구현하려고 한다.
stm32h7은 usb와 MicroSD가 장착이 되어 있고, 
FatFS로 SDMMC, SD를 활용하여 파일을 쓰고 읽을 수 있는 것을 확인한 ioc 파일을 바탕으로해서, 
fatfs방법으로 MTP를 구현하고 싶어, fileX 방법을 쓰지 않기로 했어.
첨부된 ioc를 바탕으로 내가 단계별로 무엇을 해야 할지 알려 줘
```
좋습니다. 목표는 다음과 같습니다:

* **MCU**: STM32H7B0VBT6
* **목표**: **FileX 없이**, **FatFS 기반 SD 카드**를 **Windows에서 MTP(Media Transfer Protocol) 장치로 인식되도록 구현**
* **기반**: 업로드한 `.ioc` 파일은 SD 카드에 FatFS가 정상 동작하도록 설정되어 있음

---

## ✅ 전체 구현 로드맵 (FatFS 기반 MTP with USB Full Speed)

### 1단계: STM32CubeIDE 설정 확인 및 추가 구성

**이미 완료된 상태 (업로드된 .ioc에서 확인)**:

* SDMMC1 + DMA + FATFS: OK
* USB\_OTG\_FS: 활성화되어 있음 ✅

**추가로 해야 할 설정**:

1. USB\_OTG\_FS 모드를 \*\*Device (FS)\*\*로 설정
2. USB 중간 계층에 `Middleware > USB_DEVICE`를 추가
3. Class 선택 시 **Custom Class**로 선택
4. RTOS는 사용하지 않아도 되지만, 타이머 기반 처리를 위해 필요하면 `CMSIS V2` + `FreeRTOS` 설정 (선택사항)

---

### 2단계: 코드 생성 및 USB MTP 커스텀 클래스 추가

1. `.ioc` 저장 후 코드 생성 (`Project > Generate Code`)
2. `usb_device.c`에 생성된 USB 초기화 루틴 확인
3. `usbd_customhid_if.c` 또는 `usbd_custom.c` 생성된 파일을 확인하고, 이를 MTP용으로 수정

---

### 3단계: MTP 커스텀 클래스 구현 (핵심 작업)

#### 📁 관련 파일 생성

다음 파일들을 수동으로 작성해야 합니다:

* `usbd_mtp.c`
* `usbd_mtp.h`
* `usbd_mtp_if.c`
* `mtp_storage_fatfs.c`
* `mtp_storage_fatfs.h`

#### 💡 주요 구현 내용

* `usbd_mtp.c`: USBX 없이 MTP 프로토콜의 기본 패킷 처리 (Container, Command Block, Response Block 등)
* `usbd_mtp_if.c`: Storage 연동 인터페이스 구현
* `mtp_storage_fatfs.c`: `f_open`, `f_read`, `f_write`, `f_readdir` 등 FatFS 함수 기반으로 MTP 명령에 대응
* `usbd_desc.c/h`: MTP 장치로 인식되도록 USB Descriptor 설정 변경

  * Device Class = 0x00 (Defined at Interface level)
  * Interface Class = 0x06 (Still Image)
  * Subclass = 0x01
  * Protocol = 0x01 (Picture Transfer Protocol - PTP)

---

### 4단계: Windows MTP 호환성 확보

* Windows가 STM32를 MTP 장치로 인식하도록 **정확한 USB Descriptor**를 작성
* `usbd_mtp.c`에서 USB 요청(Request) 핸들링 정확하게 구현
* Windows MTP 테스트 도구(예: MTPDetect.exe, USBView\.exe)로 장치 인식 여부 확인

---

### 5단계: 테스트 및 디버깅

* SD 카드에 기본 폴더/파일 넣기
* USB 연결 후 장치 관리자에서 `MTP Device`로 보이는지 확인
* Windows 탐색기에서 SD 카드의 파일이 보이는지 확인
* FatFS에서 파일 탐색, 읽기, 전송 모두 정상 작동하는지 확인

---

## 📌 다음으로 할 일 (지금 단계)

### 📁 1. 프로젝트에서 필요한 MTP 파일 기본 골격 제공

제가 다음 대화에서 아래 파일들 기본 코드 골격을 순차적으로 제공해드릴 수 있습니다:

* `usbd_mtp.c/.h`
* `usbd_mtp_if.c/.h`
* `mtp_storage_fatfs.c/.h`
* `usb_descriptors` 관련 수정

이후 STM32CubeIDE에 추가하고, 프로젝트에 빌드되도록 `.c/.h` 포함, USB\_DEVICE 설정 연결 등을 적용합니다.

---

## ✅ 다음 단계 질문

1. `usbd_mtp.c/.h`부터 먼저 생성해서 드릴까요?
2. 또는 `usb_descriptors` (Device/Interface 설정)부터 먼저 수정하고 싶으신가요?

원하는 순서대로 도와드릴게요.
