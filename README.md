# Converting a BMP File to Grayscale, .mem, and Sobel-Filtered Files
---
🇰🇷 개요 (Korean)

이 프로그램은 24비트(24-bit) 630×630 BMP를 입력으로 받아

8비트 그레이스케일 BMP, 2) Verilog용 .mem, 3) 소벨(Sobel) 엣지 검출 BMP를 생성합니다.

처리 파이프라인
24-bit BMP 읽기 (BGR, 아래→위 저장) 
 → 메모리상 행/열 정렬 (위→아래가 되도록 재배치)
 → 그레이스케일 변환 (0.299R + 0.587G + 0.114B)
 → ① 8-bit 팔레트 BMP 저장
 → ② .mem (픽셀당 2자리 HEX, CRLF) 저장
 → ③ Sobel 엣지 검출(Gx, Gy, sqrt), 8-bit BMP 저장

핵심 파일들 (코드와 동일)

입력: photo_630x630_centercrop.bmp (24-bit, 630×630)

출력(그레이): output_grayscale_photo_630x630_centercrop.bmp (8-bit 팔레트)

출력(엣지): output_edge_photo_630x630_centercrop.bmp (8-bit 팔레트)

출력(.mem): output_image_photo_630x630_centercrop.mem (행 우선, 상→하, 각 픽셀 2자리 16진수 + \r\n)

BMP I/O 세부

헤더 파싱: #pragma pack(push, 1)로 구조체 패딩 제거 → 디스크 레이아웃과 일치.

검증: type == 0x4D42("BM"), bitCount == 24, width/height == 630 확인.

입력 읽기: BMP는 아래 행부터 저장되므로, for (y = H-1; y >= 0; y--)로 읽어 메모리상 y=0이 상단이 되게 맞춤.

패딩: 24-bit 입력은 행 크기(3W)가 4의 배수가 되도록 패딩 존재 → 읽을 때 fseek로 건너뜀.

그레이 BMP 저장: 8-bit 팔레트(256×4, BGR0) 생성, 행 패딩은 (4 - (W % 4)) % 4.

엣지 BMP 저장: 그레이 BMP와 동일한 8-bit 팔레트/패딩 규칙.

.mem 형식 (Verilog/FPGA)

내용: 헤더 없이 픽셀 값만(0x00~0xFF) 텍스트로 저장.

순서: 상단 좌측 → 우측 (행 우선), 위 → 아래.

포맷: 각 픽셀을 2자리 대문자 HEX + \r\n (예: 7F\r\n).

용도: $readmemh로 BRAM/ROM 초기화 시 사용.

Sobel 필터 (작동 원리)

커널:

Gx = [-1 0 1; -2 0 2; -1 0 1]
Gy = [-1 -2 -1; 0 0 0; 1 2 1]


경계 처리: getPixel이 좌표를 영상 내부로 클램프 → 가장자리 복제(replicate) 방식.

기울기 계산: gx, gy 누적 → mag = sqrt(gx*gx + gy*gy) → 255로 클램프.

특징: 가중 중앙차분으로 노이즈에 비교적 강함, 연산 단순 → 임베디드/RTL 친화적.

빌드/실행 메모

포함 헤더: <stdio.h> <stdlib.h> <stdint.h> <math.h>

MSVC 사용 시 fopen_s OK. MinGW/clang이면 fopen으로 교체 가능.

리틀엔디안(x86) 가정(BMP 포맷과 일치).

입력 BMP는 압축 없음(BI_RGB) 24-bit 이어야 함.

성능/정확도 팁

가속: sqrt 대신 abs(gx)+abs(gy) 근사 사용 가능(빠름, 콘트라스트 약간 달라짐).

이진 엣지맵: 필요 시 임계값(예: Otsu 또는 비율)으로 binarize.

청소 포인트: 그레이스케일 변환 루프가 두 번 돌고 있음(기능엔 문제 없음). 한 번만 수행해도 됨.

🇺🇸 Overview (English)

This program takes a 24-bit 630×630 BMP and produces:

8-bit grayscale BMP, 2) a Verilog .mem file, and 3) a Sobel edge-detected BMP.

Processing pipeline
Read 24-bit BMP (BGR, bottom-up on disk)
 → Reorder into top-to-bottom in memory
 → Grayscale (0.299R + 0.587G + 0.114B)
 → ① Save 8-bit palette BMP
 → ② Save .mem (2-digit HEX per pixel, CRLF)
 → ③ Apply Sobel (Gx, Gy, sqrt), save 8-bit BMP

Key filenames (as in code)

Input: photo_630x630_centercrop.bmp

Grayscale BMP: output_grayscale_photo_630x630_centercrop.bmp

Edge BMP: output_edge_photo_630x630_centercrop.bmp

MEM: output_image_photo_630x630_centercrop.mem

BMP I/O specifics

Struct layout: #pragma pack(push, 1) to match on-disk headers exactly.

Validation: type == 0x4D42 ("BM"), bitCount == 24, width/height == 630.

Input read: BMP rows are stored bottom-up; read with y = H-1 … 0 so memory becomes top-down.

Padding: 24-bit rows padded to 4-byte boundaries; skip with fseek.

Saving grayscale: build a 256×4 BGR0 palette; 8-bit row padding (4 - (W % 4)) % 4.

Saving edge: same 8-bit header/palette/padding as grayscale.

.mem format (for Verilog)

Content: pixel values only (no headers), 0x00–0xFF as text.

Order: row-major from top-left to bottom-right.

Line format: 2 uppercase HEX digits + \r\n per pixel (e.g., 7F\r\n).

Use: Initialize BRAM/ROM via $readmemh.

Sobel filter (principle)

Kernels:

Gx = [-1 0 1; -2 0 2; -1 0 1]
Gy = [-1 -2 -1; 0 0 0; 1 2 1]


Border handling: getPixel clamps coordinates → replicate edges.

Gradient: accumulate gx, gy, compute mag = sqrt(gx*gx + gy*gy), clamp to 255.

Why Sobel: mild smoothing + central difference → noise-robust and RTL-friendly.

Build/Run notes

Includes: <stdio.h> <stdlib.h> <stdint.h> <math.h>

MSVC uses fopen_s; for GCC/Clang you can switch to fopen.

Assumes little-endian host; input BMP must be uncompressed 24-bit.

Performance/cleanup tips

Speedup: use abs(gx) + abs(gy) as a fast magnitude approximation.

Optional: threshold the magnitude to create a binary edge map.

Cleanup: the grayscale loop runs twice; you can remove the second pass.
