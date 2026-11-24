cd "C:\Users\rnfod\Downloads\Rocycle-main"

& {
    $ErrorActionPreference = "Stop"
    $root = (Get-Location).Path
    $source = Join-Path $root "Rocycle-main"
    $output = Join-Path $root "README.md"
    $fence = ([string][char]96) * 3

    # 01. 프로젝트 경로 확인

    if (-not (Test-Path $source)) {
        throw "Source folder not found."
    }

    # 02. 전체 README 추출 및 날짜별 정렬
    # 중복 제거 및 내용 요약 없음

    $files = @(Get-ChildItem -LiteralPath $source `
        -Recurse -File -Force -Filter "README.md" |
        Sort-Object LastWriteTime, FullName)

    if ($files.Count -eq 0) {
        throw "README files not found."
    }

    Write-Host "Total README files: $($files.Count)" `
        -ForegroundColor Cyan

    # 03. 원본 전체 출력

    foreach ($file in $files) {
        Write-Output "========================================"
        Write-Output "FILE: $($file.FullName)"
        Write-Output "DATE: $($file.LastWriteTime.ToString('yyyy-MM-dd HH:mm:ss'))"
        Write-Output "========================================"

        Get-Content -LiteralPath $file.FullName `
            -Raw -Encoding UTF8
    }

    # 04. GitHub 상대경로 생성

    function Get-RepoPath {
        param([string]$Path)

        return $Path.Substring(
            $root.Length + 1
        ).Replace('\', '/')
    }

    function Get-RepoLink {
        param([string]$Path)

        return ((Get-RepoPath $Path).Split('/') |
            ForEach-Object {
                [Uri]::EscapeDataString($_)
            }) -join '/'
    }

    # 05. 최상위 README 표지

    $readme = @'
# Rocycle - AI Recycling Sorting Robot

> AI 기반 재활용품 자동 분류 협동로봇 시스템

![ROS2](https://img.shields.io/badge/ROS2-Jazzy-22314E)
![Python](https://img.shields.io/badge/Python-3776AB)
![YOLO](https://img.shields.io/badge/YOLO12s-Object_Detection-00BFFF)
![OpenCV](https://img.shields.io/badge/OpenCV-Vision-5C3EE8)
![Kalman](https://img.shields.io/badge/Kalman_Filter-Tracking-orange)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1)
![Docker](https://img.shields.io/badge/Docker-Container-2496ED)

---

## Project Overview

Rocycle 프로젝트의 전체 개발 문서를 날짜순으로
통합한 기술 개발 기록입니다.

초기 버전부터 Final 버전까지 기존 README의
전체 본문을 생략하거나 요약하지 않고 수록합니다.

YOLO 모델 학습 및 평가 결과, 시스템 구성,
웹 관제, Docker 환경, 설계 변경 이력,
인수인계 문서를 포함합니다.

> 날짜는 원본 파일의 수정 시간을 기준으로 합니다.
> 실제 개발일과 파일 수정일은 다를 수 있습니다.

---

## Documentation Timeline

'@

    $readme += "`n**Total README Documents: $($files.Count)**`n`n"

    # 06. 날짜별 전체 목차

    $readme += "| No. | Modified Date | Document | Original |`n"
    $readme += "|---:|---|---|---|`n"

    $index = 0

    foreach ($file in $files) {
        $index++

        $date = $file.LastWriteTime.ToString(
            "yyyy-MM-dd HH:mm:ss"
        )

        $relative = Get-RepoPath $file.FullName
        $link = Get-RepoLink $file.FullName

        $readme += "| $index | $date | [Document $index](#document-$index) | [$relative]($link) |`n"
    }

    # 07. YOLO 성능 자료 추가 함수

    function Get-YoloSection {
        param([string]$ModelDir)

        $version = Split-Path $ModelDir -Leaf
        $section = "`n`n---`n`n"
        $section += "### YOLO12s $version - Model Artifacts`n`n"

        # 모델 관련 전체 파일 목록

        $artifacts = @(Get-ChildItem `
            -LiteralPath $ModelDir -Recurse -File |
            Sort-Object LastWriteTime, FullName)

        $section += "| Modified Date | Artifact | Size (MB) |`n"
        $section += "|---|---|---:|`n"

        foreach ($artifact in $artifacts) {

            $date = $artifact.LastWriteTime.ToString(
                "yyyy-MM-dd HH:mm:ss"
            )

            $link = Get-RepoLink $artifact.FullName
            $size = [Math]::Round(
                $artifact.Length / 1MB, 3
            )

            $section += "| $date | [$($artifact.Name)]($link) | $size |`n"
        }

        # 모든 모델 결과 이미지 삽입

        $images = @($artifacts | Where-Object {
            $_.Extension -match '^\.(png|jpg|jpeg|gif|svg|webp)$'
        })

        foreach ($img in $images) {

            $date = $img.LastWriteTime.ToString(
                "yyyy-MM-dd HH:mm:ss"
            )

            $link = Get-RepoLink $img.FullName

            $section += "`n`n#### $($img.Name)`n`n"
            $section += "**Modified Date:** $date`n`n"
            $section += "![$($img.Name)]($link)`n"
        }

        # 모든 JSON, YAML, CSV, TXT 결과 원문 삽입

        $textFiles = @($artifacts | Where-Object {
            $_.Extension -match '^\.(json|yaml|yml|csv|txt)$'
        })

        foreach ($item in $textFiles) {

            $date = $item.LastWriteTime.ToString(
                "yyyy-MM-dd HH:mm:ss"
            )

            $link = Get-RepoLink $item.FullName

            $language = switch ($item.Extension) {
                ".json" { "json" }
                ".yaml" { "yaml" }
                ".yml"  { "yaml" }
                ".csv"  { "csv" }
                default { "text" }
            }

            $content = Get-Content `
                -LiteralPath $item.FullName `
                -Raw -Encoding UTF8

            $section += "`n`n#### $($item.Name)`n`n"
            $section += "**Modified Date:** $date`n`n"
            $section += "[Original File]($link)`n`n"
            $section += "$fence$language`n"
            $section += $content
            $section += "`n$fence`n"
        }

        return $section
    }

    # 08. README 원본 전체를 날짜순으로 통합
    # 중복 문서도 그대로 유지
    # 본문 요약 및 삭제 없음
    # 접기 기능 사용하지 않음

    $readme += "`n`n---`n"
    $readme += "`n## Full Development Documentation`n"

    $index = 0
    $modelAdded = @{}

    foreach ($file in $files) {

        $index++

        $date = $file.LastWriteTime.ToString(
            "yyyy-MM-dd HH:mm:ss"
        )

        $relative = Get-RepoPath $file.FullName
        $link = Get-RepoLink $file.FullName

        $readme += "`n`n---`n"
        $readme += "`n## Document $index`n`n"
        $readme += "**Modified Date:** $date`n`n"
        $readme += "**Original Path:** [$relative]($link)`n`n"
        $readme += "---`n`n"

        # 원본 README 전체 본문

        $content = Get-Content `
            -LiteralPath $file.FullName `
            -Raw -Encoding UTF8

        $readme += $content
        $readme += "`n`n"

        # 모델 README 바로 아래에 실제 YOLO 결과 삽입

        if ($relative -match '/models/v[^/]+/README\.md$') {

            $modelDir = $file.DirectoryName

            if (-not $modelAdded.ContainsKey($modelDir)) {

                $readme += Get-YoloSection `
                    -ModelDir $modelDir

                $modelAdded[$modelDir] = $true
            }
        }

        Write-Host "Included: $relative" `
            -ForegroundColor Green
    }

    # 09. README가 없는 모델 버전도 누락 없이 추가

    $remainingModels = @(
        Get-ChildItem -LiteralPath $source `
            -Recurse -Directory |
        Where-Object {
            $_.Parent.Name -eq "models" -and
            $_.Name -match '^v\d'
        } |
        Sort-Object LastWriteTime, FullName
    )

    foreach ($modelDir in $remainingModels) {

        if (-not $modelAdded.ContainsKey(
            $modelDir.FullName
        )) {

            $date = $modelDir.LastWriteTime.ToString(
                "yyyy-MM-dd HH:mm:ss"
            )

            $readme += "`n`n---`n"
            $readme += "`n## Additional YOLO Model Archive`n`n"
            $readme += "**Modified Date:** $date`n`n"

            $readme += Get-YoloSection `
                -ModelDir $modelDir.FullName

            $modelAdded[$modelDir.FullName] = $true
        }
    }

    # 10. YOLO 그래프 생성 방법

    $readme += @'

---

## YOLO Training and Evaluation Guide

다음은 Ultralytics YOLO의 학습 및 검증 그래프를
생성하는 방법을 설명하기 위한 예시입니다.

실제 프로젝트 학습 설정은 위에 수록된 원본
args.yaml 및 Colab Notebook을 참고합니다.

### Training

```python
from ultralytics import YOLO

model = YOLO("yolo12s.pt")

model.train(
    data="data.yaml",
    epochs=50,
    imgsz=640,
    plots=True,
    project="runs",
    name="rocycle_train"
)
```

### Evaluation

```python
from ultralytics import YOLO

model = YOLO(
    "runs/rocycle_train/weights/best.pt"
)

metrics = model.val(
    data="data.yaml",
    split="test",
    conf=0.25,
    plots=True,
    project="runs",
    name="rocycle_test"
)

print(metrics.results_dict)
```

### Evaluation Artifacts

| File | Description |
|---|---|
| results.png | 학습 성능 그래프 |
| results.csv | Epoch별 학습 수치 |
| confusion_matrix.png | 혼동 행렬 |
| confusion_matrix_normalized.png | 정규화 혼동 행렬 |
| args.yaml | 학습 설정 |
| best.pt | 모델 가중치 |

객체 검출 모델의 평가 성능과 실제 로봇의
재활용품 분류 성공률은 서로 다른 지표입니다.

---

## Original README Archive

'@

    # 11. 모든 원본 README 링크

    $readme += "`n| Modified Date | Original README |`n"
    $readme += "|---|---|`n"

    foreach ($file in $files) {

        $date = $file.LastWriteTime.ToString(
            "yyyy-MM-dd HH:mm:ss"
        )

        $relative = Get-RepoPath $file.FullName
        $link = Get-RepoLink $file.FullName

        $readme += "| $date | [$relative]($link) |`n"
    }

    $readme += "`n---`n"
    $readme += "`n**Rocycle - AI Recycling Sorting Robot**`n"

    # 12. 기존 최상위 README 백업

    if (Test-Path $output) {

        $backup = Join-Path $env:TEMP (
            "Rocycle_README_" +
            (Get-Date -Format "yyyyMMdd_HHmmss") +
            ".md"
        )

        Copy-Item -LiteralPath $output `
            -Destination $backup

        Write-Host "Backup: $backup"
    }

    # 13. 최상위 README 생성

    [System.IO.File]::WriteAllText(
        $output,
        $readme,
        [System.Text.UTF8Encoding]::new($false)
    )

    # 14. 전체 문서 포함 여부 검증

    $missing = @()

    foreach ($file in $files) {

        $original = Get-Content `
            -LiteralPath $file.FullName `
            -Raw -Encoding UTF8

        if ($original -and -not $readme.Contains($original)) {
            $missing += $file.FullName
        }
    }

    if ($missing.Count -gt 0) {
        $missing
        throw "Original README content verification failed."
    }

    Write-Host ""
    Write-Host "========================================"
    Write-Host "README GENERATION REPORT"
    Write-Host "========================================"
    Write-Host "Original README count: $($files.Count)"
    Write-Host "Included README count: $index"
    Write-Host "Removed README count: 0"
    Write-Host "Summarized README count: 0"
    Write-Host "YOLO model directories: $($modelAdded.Count)"
    Write-Host "Missing original documents: $($missing.Count)"
    Write-Host "README size: $((Get-Item $output).Length) bytes"
    Write-Host "Original content verification: PASS" `
        -ForegroundColor Green

    # 15. 업로드 전 확인

    notepad $output

    $confirm = Read-Host "Upload complete README? (Y/N)"

    if ($confirm -ne "Y") {
        Write-Host "README saved locally. Upload cancelled."
        return
    }

    # 16. 최상위 README만 GitHub에 업로드

    git add -- README.md

    if ($LASTEXITCODE -ne 0) {
        throw "Git add failed."
    }

    git diff --cached --quiet -- README.md

    if ($LASTEXITCODE -eq 1) {

        git commit --only `
            -m "Remake full chronological README with original documentation and YOLO results" `
            -- README.md

        if ($LASTEXITCODE -ne 0) {
            throw "Git commit failed."
        }
    }
    elseif ($LASTEXITCODE -ne 0) {
        throw "Git diff failed."
    }

    git push origin main

    if ($LASTEXITCODE -ne 0) {
        throw "Git push failed."
    }

    # 17. GitHub 저장소 열기

    gh repo view gmltjd0725-hue/yolo-main --web

    Write-Host "README upload completed." `
        -ForegroundColor Green
}
