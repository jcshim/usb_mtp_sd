STM32CubeIDE 1.18.1을 사용 중이며, STM32H7B0VBT6 MCU 기반 보드를 사용합니다.

현재 SDMMC1 + FatFs 구성이 완료된 .ioc 파일(예: STM32H7B0VBT6 SD FatFs.ioc)을 가지고 있으며,
이제 USB Full Speed 모드(PA11/PA12 사용, 외부 PHY 없음)에서 MTP(Media Transfer Protocol)를 적용하여,
Windows PC에서 STM32 보드를 외장 저장 장치처럼 인식하고,
SD 카드의 파일을 MTP를 통해 읽고 쓸 수 있게 만들고자 합니다.

저는 임베디드 USB 및 MTP 관련 지식이 부족한 초보자입니다.
그래서 전체 작업을 오늘 내로 완료하고 싶습니다.

다음과 같은 모든 자료를 포함해서 완성된 상태로 제공해 주세요:

1. 수정/확장된 .ioc 파일 (USBX + Custom MTP Class 포함)
2. 필요한 모든 소스코드 (USBX, MTP, FatFs 연동부, ux_device_class_mtp.c 등)
3. Windows 10 이상에서 MTP 장치로 인식되도록 설정된 Descriptor 파일들 (ux_device_descriptor.h 등)
4. MTP 명령이 SD카드의 FatFs 파일 시스템과 연결되어 실제 동작하도록 구현된 코드
5. 프로젝트 생성 후 바로 빌드가 가능하도록 구성된 STM32CubeIDE 프로젝트 폴더 전체

그리고 마지막으로:

✅ **모든 파일이 제공된 후, 제가 해야 할 단계별 작업 (빌드 방법, 보드 연결, 실행 확인법, Windows에서 인식되는지 확인하는 절차)**까지  
초보자도 따라 할 수 있게 **순서대로 친절하게 설명해 주세요.**

저는 이 프로젝트를 **오늘 반드시 완성하고 싶습니다.**

가능하시다면 빠르게 도와주시면 정말 감사하겠습니다.
