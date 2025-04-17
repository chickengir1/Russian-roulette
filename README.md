## 4.1 CSV 로드 & 초기 프레임 렌더링

### 4.1‑① CSV 다운로드 요청 → 응답 수신 (1 – 7)

```mermaid
sequenceDiagram
    participant Test2Page as Test2 페이지 컴포넌트
    participant LoaderHook as useChunkedCSVLoader
    participant LoadingHook as useLoading
    participant ProgressComp as ProgressIndicator 컴포넌트
    participant LoadingUtils as loadingUtils
    participant Browser API

    Test2Page->>LoaderHook: 1. 초기화 (CSV_DATA_PATH 전달)
    LoaderHook->>LoadingHook: 2. 초기화 (loadingHandler 생성)
    Test2Page->>ProgressComp: 3. 초기 로딩 상태 표시 (loading=true)
    LoaderHook->>LoadingUtils: 4. downloadCSV(path, loadingHandler)
    LoadingUtils->>Browser API: 5. fetch(path)
    Browser API-->>LoadingUtils: 6. CSV 텍스트 응답
    LoadingUtils-->>LoaderHook: 7. CSV 텍스트 반환 (진행률 업데이트)

```

### 4.1‑② CSV 파싱 → 온도 데이터 추출 (8 – 12)

```mermaid
sequenceDiagram
    participant LoaderHook as useChunkedCSVLoader
    participant ParserUtil as csvParser
    participant LoaderUtils as csvLoaderUtils

    LoaderHook->>ParserUtil: 8. parseCSV(csvText)
    ParserUtil-->>LoaderHook: 9. 파싱된 행 (string[]) 반환
    LoaderHook->>LoaderUtils: 10. extractTemperatureData(rows, loadingHandler)
    LoaderUtils->>ParserUtil: 11. extractTemperatureValue 반복 호출
    LoaderUtils-->>LoaderHook: 12. 온도 데이터 배열 및 min/max Temp 반환 (진행률 업데이트)

```

### 4.1‑③ 프레임 생성 & 로딩 완료 (13 – 16)

```mermaid
sequenceDiagram
    participant LoaderHook as useChunkedCSVLoader
    participant FrameUtils as temperatureFrameUtils
    participant LoadingHook as useLoading
    participant Test2Page as Test2 페이지 컴포넌트

    LoaderHook->>FrameUtils: 13. generateFrames(temperatureData, loadingHandler)
    FrameUtils-->>LoaderHook: 14. 프레임 배열 (number[][]) 반환 (진행률 업데이트)
    LoaderHook->>LoadingHook: 15. 로딩 완료 상태 설정 (loading=false)
    LoaderHook-->>Test2Page: 16. 최종 데이터 반환 { frames, minTemp, maxTemp }

```

### 4.1‑④ 렌더러 초기화 & 첫 프레임 표시 (17 – 25)

```mermaid
sequenceDiagram
    participant Test2Page as Test2 페이지 컴포넌트
    participant PlayerHook as useFramePlayer
    participant ColorMapHook as useColorMap
    participant RendererHook as useCanvasRenderer
    participant CanvasComp as ThermalCanvas 컴포넌트
    participant pixelUtils as pixelUtils
    participant User as 사용자

    Test2Page->>PlayerHook: 17. 초기화 (frameCount 전달)
    PlayerHook-->>Test2Page: 18. currentFrame=0 반환
    Test2Page->>ColorMapHook: 19. 초기화 (minTemp, maxTemp 전달)
    ColorMapHook-->>Test2Page: 20. colorMap 함수 반환
    Test2Page->>RendererHook: 21. 초기화 (canvasRef, firstFrameData, width, height, colorMap 전달)
    RendererHook->>CanvasComp: 22. getContext('2d') 및 createImageData()
    RendererHook->>pixelUtils: 23. mapPixelColors, applyPixelColors 호출
    RendererHook->>CanvasComp: 24. putImageData(imageData)
    CanvasComp-->>User: 25. 첫 프레임 이미지 표시

```

---

## 4.2 프레임 재생 컨트롤 (재생/정지)

### 4.2‑① Play 버튼 처리 & isPlaying 활성화 (1 – 4)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant FrameControlsComp as FrameControls 컴포넌트
    participant Test2Page as Test2 페이지 컴포넌트
    participant PlayerHook as useFramePlayer

    User->>FrameControlsComp: 1. 재생(Play) 버튼 클릭
    FrameControlsComp->>Test2Page: 2. onPlayToggle() 호출
    Test2Page->>Test2Page: 3. isPlaying=true
    Test2Page->>PlayerHook: 4. isPlaying=true 전달 (useEffect 시작)

```

### 4.2‑② FPS 루프‑프레임 업데이트 & 렌더 (5 – 9)

```mermaid
sequenceDiagram
    participant PlayerHook as useFramePlayer
    participant Test2Page as Test2 페이지 컴포넌트
    participant RendererHook as useCanvasRenderer
    participant CanvasComp as ThermalCanvas 컴포넌트
    participant pixelUtils as pixelUtils

    loop FPS 간격마다
        PlayerHook->>PlayerHook: 5. currentFrame 업데이트
        PlayerHook-->>Test2Page: 6. currentFrame 값 전달
        Test2Page->>RendererHook: 7. currentFrameData 전달
        RendererHook->>pixelUtils: 8. filterChangedPixels, applyPixelColors 호출
        RendererHook->>CanvasComp: 9. putImageData(updatedImageData)
    end

```

### 4.2‑③ Pause 버튼 처리 & 루프 정지 (10 – 13)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant FrameControlsComp as FrameControls 컴포넌트
    participant Test2Page as Test2 페이지 컴포넌트
    participant PlayerHook as useFramePlayer

    User->>FrameControlsComp: 10. 정지(Pause) 버튼 클릭
    FrameControlsComp->>Test2Page: 11. onPlayToggle() 호출
    Test2Page->>Test2Page: 12. isPlaying=false
    Test2Page->>PlayerHook: 13. isPlaying=false 전달 (clearInterval)

```

---

## 4.3 이미지 크롭 & 스냅샷 생성

### 4.3‑① Crop 모드 진입 (1 – 4)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant FrameControlsComp as FrameControls 컴포넌트
    participant Test2Page as Test2 페이지 컴포넌트
    participant CanvasComp as ThermalCanvas 컴포넌트

    User->>FrameControlsComp: 1. '크롭 모드' 클릭
    FrameControlsComp->>Test2Page: 2. onCropToggle() 호출
    Test2Page->>Test2Page: 3. cropMode=true, isPlaying=false
    Test2Page->>CanvasComp: 4. cropMode=true 전달

```

### 4.3‑② 드래그·영역 미리보기 (5 – 7)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant CanvasComp as ThermalCanvas 컴포넌트
    participant SnapshotHook as useSnapshotCreator

    User->>CanvasComp: 5. 캔버스 드래그
    CanvasComp->>CanvasComp: 6. 드래그 영역 계산·미리보기
    CanvasComp->>SnapshotHook: 7. onCropSelected(finalCropArea)

```

### 4.3‑③ 임시 캔버스 → 통계 계산 (8 – 13)

```mermaid
sequenceDiagram
    participant SnapshotHook as useSnapshotCreator
    participant Test2Page as Test2 페이지 컴포넌트
    participant Browser API
    participant CanvasComp as ThermalCanvas 컴포넌트
    participant SnapshotUtil as snapShotUtils

    SnapshotHook->>Test2Page: 8. setIsPlaying(false)
    SnapshotHook->>Browser API: 9. 임시 캔버스 생성
    SnapshotHook->>CanvasComp: 10. getCanvasElement()
    SnapshotHook->>Browser API: 11. drawImage(mainCanvas, cropArea)
    SnapshotHook->>SnapshotUtil: 12. calculateCroppedStats(...)
    SnapshotUtil-->>SnapshotHook: 13. min/max/avg 반환

```

### 4.3‑④ DataURL 생성 & snapshots 업데이트 (14 – 21)

```mermaid
sequenceDiagram
    participant SnapshotHook as useSnapshotCreator
    participant DateUtil as dateUtils
    participant Browser API
    participant Test2Page as Test2 페이지 컴포넌트

    SnapshotHook->>DateUtil: 14. getFormattedTimestamp()
    DateUtil-->>SnapshotHook: 15. 시간 문자열
    SnapshotHook->>Browser API: 16. 임시 캔버스 toDataURL
    Browser API-->>SnapshotHook: 17. 이미지 DataURL
    SnapshotHook->>Test2Page: 18. setSnapshots([...prev, newSnapshot])
    Test2Page->>Test2Page: 19. snapshots 상태 업데이트
    SnapshotHook->>Test2Page: 20. setCropMode(false)
    Test2Page->>Test2Page: 21. cropMode=false

```

---

## 4.4 스냅샷 카드 다운로드

### 4.4‑① DOM → Canvas 렌더 (1 – 6)

```mermaid
sequenceDiagram
    participant User as 사용자
    participant SnapshotListComp as SnapshotList 컴포넌트
    participant BrowserAPI as Browser API
    participant Html2CanvasLib as html2canvas

    User->>SnapshotListComp: 1. '다운로드' 버튼 클릭
    SnapshotListComp->>SnapshotListComp: 2. downloadSnapshotCard(timestamp)
    SnapshotListComp->>BrowserAPI: 3. '.snapshot-card' 요소 찾기
    SnapshotListComp->>Html2CanvasLib: 4. html2canvas(cardElement)
    Html2CanvasLib->>BrowserAPI: 5. DOM 분석·렌더링
    Html2CanvasLib-->>SnapshotListComp: 6. 캔버스 객체

```

### 4.4‑② DataURL → PNG 저장 (7 – 10)

```mermaid
sequenceDiagram
    participant SnapshotListComp as SnapshotList 컴포넌트
    participant BrowserAPI as Browser API
    participant User as 사용자

    SnapshotListComp->>BrowserAPI: 7. canvas.toDataURL('image/png')
    BrowserAPI-->>SnapshotListComp: 8. 데이터 URL
    SnapshotListComp->>BrowserAPI: 9. <a> 생성·click()
    BrowserAPI-->>User: 10. 파일 저장 프롬프트

```

---

## 4.5 온도 데이터 차트 표시 (스냅샷 카드 내)

### 4.5‑① 스냅샷 전달 & 개별 차트 렌더 (1 – 2)

```mermaid
sequenceDiagram
    participant Test2Page as Test2 페이지 컴포넌트
    participant SnapshotListComp as SnapshotList 컴포넌트
    participant TempBarChartComp as TemperatureBarChart 컴포넌트

    Test2Page->>SnapshotListComp: 1. snapshots, overallMin/MaxTemp
    loop 각 snapshot
        SnapshotListComp->>TempBarChartComp: 2. stats, minTemp, maxTemp
    end

```

### 4.5‑② 막대 높이 계산 & 그리기 (3 – 4)

```mermaid
sequenceDiagram
    participant TempBarChartComp as TemperatureBarChart 컴포넌트
    participant BrowserAPI as Browser API

    TempBarChartComp->>TempBarChartComp: 3. 막대 높이 계산
    TempBarChartComp->>BrowserAPI: 4. div 막대 렌더링

```
