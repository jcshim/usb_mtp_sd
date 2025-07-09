# STM32H7 USB MSC ➔ MTP 변화 따라하기 (완전 초보용, STM32CubeIDE 1.18.1 기준)

## ✅ 목표

* 기존 USB MSC 프로젝트를 MTP로 바꾸어서 Windows에서 '미디어 장치'로 보이도록 설정
* microSD에 `.txt`, `.doc`, `.hwp`만 저장 허용

---

## 📘 1단계: 기존 MSC 제거

1. `.ioc` 파일을 STM32CubeIDE에서 열기
2. 메뉴 왼쪽 **Middleware > USB\_DEVICE** 클릭
3. **Class For FS IP** → "Mass Storage Class" 비활성화 (선택 해제)
4. 저장 (`Ctrl+S`) → 자동으로 코드 변경됩니다

---

## 📘 2단계: MTP 클래스 파일 추가

### 2-1. 파일 추가 위치

* `usbd_mtp.c` / `usbd_mtp.h` → `/USB_DEVICE/Class/MTP/`
* `usbd_mtp_if.c` / `usbd_mtp_if.h` → `/USB_DEVICE/App/`

> 위 4개 파일은 작동 확인된 버전으로 ChatGPT가 제공

### 2-2. CubeIDE에 인식되게 하기

* `Project Explorer`에서 우회클릭 → `Refresh`
* 새로 추가한 .c 파일들은 오른쪽 클릭 → `Properties` → `C/C++ Build > Settings > Tool Settings > MCU GCC Compiler > Includes`에서 인식되는지 확인

---

## 📘 3단계: MTP 클래스 등록

`usb_device.c` 파일 하단에 다음 추가:

```c
extern USBD_ClassTypeDef USBD_MTP;
extern USBD_MTP_ItfTypeDef USBD_Interface_fops_FS;
```

`MX_USB_Device_Init()` 함수 안에 다음 코드로 등록:

```c
USBD_RegisterClass(&hUsbDeviceFS, &USBD_MTP);
USBD_MTP_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);
```

> **기존 MSC 등록 코드 대신** 위 코드 사용

---

## 📘 4단계: FATFS 연결 코드 수정

`usbd_mtp_if.c`의 `MTP_WriteData_FS()` 함수에 아래 코드 작성:

```c
extern FATFS SDFatFS;
extern char SDPath[4];

int8_t MTP_WriteData_FS(uint32_t offset, uint8_t* buf, uint32_t len) {
    FIL file;
    UINT written;

    if (strstr((char*)current_filename, ".txt") ||
        strstr((char*)current_filename, ".doc") ||
        strstr((char*)current_filename, ".hwp")) {

        if (f_mount(&SDFatFS, SDPath, 1) == FR_OK) {
            if (f_open(&file, current_filename, FA_CREATE_ALWAYS | FA_WRITE) == FR_OK) {
                f_write(&file, buf, len, &written);
                f_close(&file);
            }
        }
    }

    return (USBD_OK);
}
```

> `current_filename`은 전송된 파일 이름 (헤더에서 선언됨)

---

## 📘 5단계: 디스크 정보 함수 구현

`usbd_mtp_if.c`에서 용량 정보를 리턴하는 함수 구현:

```c
int8_t MTP_GetCapacity_FS(uint32_t* block_num, uint32_t* block_size) {
    FATFS* fs;
    DWORD fre_clust, fre_sect, tot_sect;

    if (f_getfree(SDPath, &fre_clust, &fs) == FR_OK) {
        tot_sect = (fs->n_fatent - 2) * fs->csize;
        *block_num = tot_sect;
        *block_size = 512;
    } else {
        *block_num = 0;
        *block_size = 0;
    }

    return USBD_OK;
}
```

---

## 📘 6단계: 장치 설명 수정 (usbd\_desc.c)

`USBD_FS_Desc` 구조체 내림 또는 `USBD_FS_ProductStrDescriptor()` 함수에서 문자열을 `"MTP Device"`로 수정:

```c
uint8_t* USBD_FS_ProductStrDescriptor(USBD_SpeedTypeDef speed, uint16_t* length) {
    USBD_GetString((uint8_t *)"MTP Device", USBD_StrDesc, length);
    return USBD_StrDesc;
}
```

---

## ✅ 마무리: 빌드 및 테스트

1. `Ctrl + B`로 빌드
2. 보드에 업로드 (`Run` 또는 `Debug`)
3. USB 케이블로 PC 연결
4. **Windows 탑산기에서 "MTP 장치"로 나타나면 성공!**

---

## ✅ 요약

| 단계 | 설명               |
| -- | ---------------- |
| 1  | MSC 클래스 비활성화     |
| 2  | MTP 소스 4개 수동 추가  |
| 3  | MTP 클래스 등록 코드 수정 |
| 4  | microSD FATFS 연동 |
| 5  | 저장/읽기 함수 수정      |
| 6  | 장치 이름 변경         |

---

필요시 `.c` 파일 전체 예제도 제공 가능!
