cd "C:\Users\rnfod\Downloads\Rocycle-main"

& {
    $ErrorActionPreference = "Stop"
    $root = (Get-Location).Path
    $source = Join-Path $root "Rocycle-main"

    # ========================================
    # 1. 기존 README 파일 추출 및 날짜별 정렬
    # ========================================

    $files = Get-ChildItem -Path $source `
        -Recurse -File -Filter "README.md" |
        Sort-Object LastWriteTime, FullName

    if (-not $files) {
        Write-Host "README files not found." -ForegroundColor Red
        return
    }

    $files | ForEach-Object {
        Write-Output "========================================"
        Write-Output "FILE: $($_.FullName)"
        Write-Output "DATE: $($_.LastWriteTime.ToString('yyyy-MM-dd HH:mm:ss'))"
        Write-Output "========================================"
        Get-Content -LiteralPath $_.FullName -Raw -Encoding UTF8
    }

    # ========================================
    # 2. 최상위 README 표지 및 프로젝트 소개
    # ========================================

    $readme = @'
# Rocycle - AI Recycling Sorting Robot

> AI 기반 재활용품 자동 분류 협동로봇 시스템

![ROS2](https://img.shields.io/badge/ROS2-Jazzy-22314E)
![Python](https://img.shields.io/badge/Python-3.12-3776AB)
![YOLO](https://img.shields.io/badge/YOLO12s-Object_Detection-00FFFF)
![OpenCV](https://img.shields.io/badge/OpenCV-Vision-5C3EE8)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Database-4169E1)
![Docker](https://img.shields.io/badge/Docker-Container-2496ED)

---

## 01. Project Overview

재활용품 객체 검출, 이동 객체 추적, 로봇 제어,
컨베이어 연동 및 웹 관제 기능을 통합한 자동 분류 시스템입니다.

### 주요 기술

| 기술 | 적용 분야 |
|---|---|
| YOLO12s | 재활용품 객체 검출 |
| OpenCV | 영상 처리 |
| Kalman Filter | 이동 객체 추적 및 위치 예측 |
| ROS 2 | 시스템 통신 및 로봇 제어 |
| PostgreSQL | 작업 데이터 관리 |
| Web UI | 실시간 관제 |
| Docker | 실행 환경 구성 |

---

## 02. System Architecture

```mermaid
flowchart TD
    A[Camera] --> B[YOLO12s Detection]
    B --> C[Object Tracking]
    C --> D[Position Prediction]
    D --> E[Robot Control]
    E --> F[Gripper and Sorting]
    G[Conveyor] --> D
    E --> H[Database]
    H --> I[Web API]
    I --> J[Web UI]
    J --> E
