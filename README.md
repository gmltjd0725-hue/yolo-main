cd "C:\Users\rnfod\Downloads\Rocycle-main"

& {
    $ErrorActionPreference = "Stop"
    $root = (Get-Location).Path
    $source = Join-Path $root "Rocycle-main"
    $output = Join-Path $root "README.md"

    if (-not (Test-Path $source)) {
        throw "Source folder not found."
    }

    # ========================================
    # 01. 전체 README 추출
    # ========================================

    $files = @(Get-ChildItem -LiteralPath $source `
        -Recurse -File -Filter "README.md" |
        Sort-Object LastWriteTime, FullName)

    if ($files.Count -eq 0) {
        throw "README files not found."
    }

    foreach ($file in $files) {
        Write-Output "========================================"
        Write-Output "FILE: $($file.FullName)"
        Write-Output "DATE: $($file.LastWriteTime.ToString('yyyy-MM-dd HH:mm:ss'))"
        Write-Output "========================================"

        Get-Content -LiteralPath $file.FullName `
            -Raw -Encoding UTF8
    }

    # ========================================
    # 02. 상대경로 처리 함수
    # ========================================

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

    function Resolve-MarkdownLink {
        param(
            [string]$Target,
            [string]$Directory
        )

        if ($Target -match '^(?:[a-zA-Z][a-zA-Z0-9+.-]*:|#|/)') {
            return $Target
        }

        try {
            $parts = $Target -split '#', 2
            $decoded = [Uri]::UnescapeDataString($parts[0])

            $absolute = [IO.Path]::GetFullPath(
                (Join-Path $Directory $decoded)
            )

            if (-not $absolute.StartsWith(
                $root + "\",
                [StringComparison]::OrdinalIgnoreCase
            )) {
                return $Target
            }

            $result = Get-RepoLink $absolute

            if ($parts.Count -gt 1) {
                $result += "#" + $parts[1]
            }

            return $result
        }
        catch {
            return $Target
        }
    }

    # ========================================
    # 03. 원문 구조 및 링크 보정
    # ========================================

    function Format-OriginalReadme {
        param(
            [string]$Content,
            [string]$Directory
        )

        $lines = $Content -split '\r?\n'
        $result = New-Object System.Collections.Generic.List[string]

        $inCode = $false
        $fenceChar = ""
        $fenceLength = 0

        foreach ($line in $lines) {

            if ($line -match '^\s{0,3}(`{3,}|~{3,})(.*)$') {

                $marker = $Matches[1]

                if (-not $inCode) {
                    $inCode = $true
                    $fenceChar = $marker.Substring(0,1)
                    $fenceLength = $marker.Length
                }
                elseif (
                    $marker.Substring(0,1) -eq $fenceChar -and
                    $marker.Length -ge $fenceLength -and
                    $Matches[2].Trim() -eq ""
                ) {
                    $inCode = $false
                }

                $result.Add($line)
                continue
            }

            if ($inCode -or $line -match '^(    |\t)') {
                $result.Add($line)
                continue
            }

            # 제목 계층만 조정

            $line = [regex]::Replace(
                $line,
                '^(#{1,6})([ \t]+)',
                {
                    param($m)

                    $level = [Math]::Min(
                        6,
                        $m.Groups[1].Value.Length + 2
                    )

                    return ('#' * $level) +
                        $m.Groups[2].Value
                }
            )

            # Markdown 이미지 및 링크 보정

            $line = [regex]::Replace(
                $line,
                '\]\((?<url>[^)\s]+)(?<title>\s+[^)]*)?\)',
                {
                    param($m)

                    $target = $m.Groups["url"].Value

                    $newTarget = Resolve-MarkdownLink `
                        -Target $target `
                        -Directory $Directory

                    return "](" + $newTarget +
                        $m.Groups["title"].Value + ")"
                }
            )

            # HTML 이미지 경로 보정

            $line = [regex]::Replace(
                $line,
                '(?<prefix>\bsrc\s*=\s*")(?<url>[^"]+)"',
                {
                    param($m)

                    $target = Resolve-MarkdownLink `
                        -Target $m.Groups["url"].Value `
                        -Directory $Directory

                    return $m.Groups["prefix"].Value +
                        $target + '"'
                }
            )

            $result.Add($line)
        }

        # 닫히지 않은 코드 블록이 다음 문서에
        # 영향을 주지 않도록 종료 처리

        if ($inCode) {
            $result.Add($fenceChar * $fenceLength)
        }

        return ($result -join "`n")
    }

    # ========================================
    # 04. 날짜별 문서 및 YOLO 모델 수집
    # ========================================

    $entries = @()

    foreach ($file in $files) {

        $entries += [PSCustomObject]@{
            Type = "README"
            Date = $file.LastWriteTime
            Path = $file.FullName
            Name = Get-RepoPath $file.FullName
        }
    }

    $modelDirs = @(Get-ChildItem `
        -LiteralPath $source -Recurse -Directory |
        Where-Object {
            $_.Parent.Name -eq "models" -and
            $_.Name -match '^v\d'
        })

    foreach ($dir in $modelDirs) {

        $modelFiles = @(Get-ChildItem `
            -LiteralPath $dir.FullName `
            -Recurse -File)

        if ($modelFiles.Count -eq 0) {
            continue
        }

        $latest = $modelFiles |
            Sort-Object LastWriteTime -Descending |
            Select-Object -First 1

        $entries += [PSCustomObject]@{
            Type = "YOLO"
            Date = $latest.LastWriteTime
            Path = $dir.FullName
            Name = Get-RepoPath $dir.FullName
        }
    }

    $entries = @($entries |
        Sort-Object Date, Path)

    # ========================================
    # 05. README 표지
    # ========================================

    $readme = @'
# ♻️ Rocycle

## AI Recycling Sorting Robot

> AI 기반 재활용품 자동 분류 협동로봇 시스템

![ROS2](https://img.shields.io/badge/ROS2-Jazzy-22314E)
![Python](https://img.shields.io/badge/Python-3776AB)
![YOLO](https://img.shields.io/badge/YOLO12s-Object_Detection-00BFFF)
![OpenCV](https://img.shields.io/badge/OpenCV-Vision-5C3EE8)
![Kalman](https://img.shields.io/badge/Kalman_Filter-Tracking-orange)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1)
![Docker](https://img.shields.io/badge/Docker-2496ED)

---

## Project Overview

Rocycle 프로젝트의 초기 개발 문서부터 Final 버전까지
전체 README를 날짜순으로 정리한 개발 기록입니다.

기존 문서의 기술 설명, 코드, 실행 방법,
설계 내용 및 개발 기록을 유지합니다.

> 날짜는 파일 수정 시간을 기준으로 하며,
> 실제 개발 날짜와 다를 수 있습니다.

---

## Documentation Timeline

'@

    $readme += "`n- Original README documents: $($files.Count)"
    $readme += "`n- YOLO model directories: $($modelDirs.Count)"
    $readme += "`n- Total timeline entries: $($entries.Count)`n`n"

    # ========================================
    # 06. 날짜별 목차
    # ========================================

    $readme += "| No. | Modified Date | Type | Document |`n"
    $readme += "|---:|---|---|---|`n"

    $index = 0

    foreach ($entry in $entries) {

        $index++
        $date = $entry.Date.ToString("yyyy-MM-dd")
        $name = $entry.Name

        $readme += "| $index | $date | $($entry.Type) | [$name](#record-$index) |`n"
    }

    # ========================================
    # 07. 날짜별 전체 본문 통합
    # ========================================

    $index = 0
    $includedReadmes = 0
    $includedModels = 0

    foreach ($entry in $entries) {

        $index++

        $date = $entry.Date.ToString(
            "yyyy-MM-dd HH:mm:ss"
        )

        $link = Get-RepoLink $entry.Path

        $readme += "`n`n---`n`n"
        $readme += "## Record $index`n`n"
        $readme += "**Date:** $date`n`n"
        $readme += "**Source:** [$($entry.Name)]($link)`n`n"

        if ($entry.Type -eq "README") {

            $file = Get-Item -LiteralPath $entry.Path

            $original = Get-Content `
                -LiteralPath $file.FullName `
                -Raw -Encoding UTF8

            $formatted = Format-OriginalReadme `
                -Content $original `
                -Directory $file.DirectoryName

            $readme += $formatted
            $readme += "`n`n"

            $includedReadmes++

            Write-Host "README included: $($entry.Name)" `
                -ForegroundColor Green
        }

        # ====================================
        # 08. YOLO 학습 및 검증 자료
        # ====================================

        elseif ($entry.Type -eq "YOLO") {

            $version = Split-Path $entry.Path -Leaf

            $readme += "### YOLO Model - $version`n`n"

            $artifacts = @(Get-ChildItem `
                -LiteralPath $entry.Path `
                -Recurse -File |
                Sort-Object LastWriteTime, FullName)

            # 학습 결과 그래프

            $trainingImages = @($artifacts |
                Where-Object {
                    $_.Name -eq "results.png"
                })

            foreach ($img in $trainingImages) {

                $imageLink = Get-RepoLink $img.FullName

                $readme += "#### Training Results`n`n"
                $readme += "![$($img.Name)]($imageLink)`n`n"
                $readme += "학습 및 검증 Loss, Precision, Recall, mAP 변화를 확인하는 원본 그래프입니다.`n`n"
            }

            # 혼동 행렬

            $matrixImages = @($artifacts |
                Where-Object {
                    $_.Name -match '^confusion_matrix.*\.(png|jpg|jpeg)$'
                })

            foreach ($img in $matrixImages) {

                $imageLink = Get-RepoLink $img.FullName

                $readme += "#### $($img.BaseName)`n`n"
                $readme += "![$($img.Name)]($imageLink)`n`n"
            }

            # 나머지 모델 결과 이미지

            $otherImages = @($artifacts |
                Where-Object {
                    $_.Extension -match '^\.(png|jpg|jpeg|gif|svg|webp)$' -and
                    $_.Name -ne "results.png" -and
                    $_.Name -notmatch '^confusion_matrix'
                })

            foreach ($img in $otherImages) {

                $imageLink = Get-RepoLink $img.FullName

                $readme += "#### $($img.Name)`n`n"
                $readme += "![$($img.Name)]($imageLink)`n`n"
            }

            # 실제 검증 성능 JSON 전체

            foreach ($metric in ($artifacts |
                Where-Object {
                    $_.Name -eq "test_metrics.json"
                })) {

                $data = Get-Content `
                    -LiteralPath $metric.FullName `
                    -Raw -Encoding UTF8

                $fence = ([string][char]96) * 3

                $readme += "#### Test Metrics`n`n"
                $readme += "$fence" + "json`n"
                $readme += $data
                $readme += "`n$fence`n`n"
            }

            # 모델 관련 원본 파일 전체 연결

            $readme += "#### Model Artifacts`n`n"
            $readme += "| Date | File | Size (MB) |`n"
            $readme += "|---|---|---:|`n"

            foreach ($artifact in $artifacts) {

                $artifactDate = $artifact.LastWriteTime.ToString(
                    "yyyy-MM-dd"
                )

                $artifactLink = Get-RepoLink $artifact.FullName

                $size = [Math]::Round(
                    $artifact.Length / 1MB, 3
                )

                $readme += "| $artifactDate | [$($artifact.Name)]($artifactLink) | $size |`n"
            }

            $includedModels++

            Write-Host "YOLO included: $($entry.Name)" `
                -ForegroundColor Cyan
        }
    }

    # ========================================
    # 09. YOLO 그래프 생성 방법
    # ========================================

    $readme += @'

---

## YOLO Training and Evaluation Guide

아래 코드는 YOLO 학습 및 평가 그래프를 생성하는
방법을 설명하기 위한 예시입니다.

실제 프로젝트 설정은 각 모델의 args.yaml 및
Colab Notebook을 참고합니다.

### Model Training

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

### Model Evaluation

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

### Generated Files

| File | Description |
|---|---|
| results.png | 학습 성능 그래프 |
| results.csv | Epoch별 학습 수치 |
| confusion_matrix.png | 혼동 행렬 |
| confusion_matrix_normalized.png | 정규화 혼동 행렬 |
| args.yaml | 학습 설정 |
| best.pt | 모델 가중치 |

---

## Original Documentation Archive

'@

    # ========================================
    # 10. 전체 원본 문서 목록
    # ========================================

    $readme += "`n| Date | Original README |`n"
    $readme += "|---|---|`n"

    foreach ($file in $files) {

        $date = $file.LastWriteTime.ToString("yyyy-MM-dd")
        $relative = Get-RepoPath $file.FullName
        $link = Get-RepoLink $file.FullName

        $readme += "| $date | [$relative]($link) |`n"
    }

    # ========================================
    # 11. 문서 포함 개수 검증
    # ========================================

    if ($includedReadmes -ne $files.Count) {
        throw "README count verification failed."
    }

    # ========================================
    # 12. 기존 README 백업
    # ========================================

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

    # ========================================
    # 13. 새로운 README 생성
    # ========================================

    [System.IO.File]::WriteAllText(
        $output,
        $readme,
        [System.Text.UTF8Encoding]::new($false)
    )

    Write-Host ""
    Write-Host "========================================"
    Write-Host "README GENERATION REPORT"
    Write-Host "========================================"
    Write-Host "Original README count: $($files.Count)"
    Write-Host "Included README count: $includedReadmes"
    Write-Host "Removed README count: 0"
    Write-Host "Summarized README count: 0"
    Write-Host "Included YOLO sections: $includedModels"
    Write-Host "Total timeline entries: $($entries.Count)"
    Write-Host "README size: $((Get-Item $output).Length) bytes"
    Write-Host "Document count verification: PASS" `
        -ForegroundColor Green

    if ((Get-Item $output).Length -gt 500000) {
        Write-Warning "README is large. GitHub may truncate its rendered view."
    }

    # ========================================
    # 14. 업로드 전 검토
    # ========================================

    notepad $output

    $confirm = Read-Host "Upload new README? (Y/N)"

    if ($confirm -ne "Y") {
        Write-Host "README saved locally."
        return
    }

    # ========================================
    # 15. GitHub 업로드
    # ========================================

    git add -- README.md

    if ($LASTEXITCODE -ne 0) {
        throw "Git add failed."
    }

    git diff --cached --quiet -- README.md

    if ($LASTEXITCODE -eq 1) {

        git commit --only `
            -m "Rebuild chronological README and preserve original documentation" `
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

    gh repo view gmltjd0725-hue/yolo-main --web

    Write-Host "README upload completed." `
        -ForegroundColor Green
}
