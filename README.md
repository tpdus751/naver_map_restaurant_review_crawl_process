# 네이버 지도 음식점 리뷰 크롤링 방법 설명
네이버 지도에서 음식점 리뷰를 크롤링 하는 과정에 대하여 상세하게 다룹니다.

## 크롤링을 한 이유
2학년 1학기 웹크롤링실습 강의 기말고사 평가 항목으로 감성분석 모델을 활용하여 웹 프로젝트를 진행하는 "성남시 음식점 리뷰 감성분석 및 분석 웹 서비스" 에 모델 학습을 위한 데이터 준비 과정으로 필요하였고,
"내돈내픽" 서비스에서 리뷰를 수집하는 과정이 필요하여 두 개의 프로젝트에 맞게 다른 목적으로 크롤링을 진행하게 되었음.

## requirements
필요한 라이브러리 : selenium, webdriver_manager (크롬 브라우저 사전에 설치되어 있어야 함)

## 진행 프로세스 별 상세 코드 설명 및 작동 결과
### 0. 전역 설정 및 임포트
```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import NoSuchElementException
from webdriver_manager.chrome import ChromeDriverManager
import time, re

flag = True
```
Selenium, re, time, webdriver-manager 등 크롤링 필수 패키지 임포트

flag: 전처리 중 주소 무효 처리 여부를 결정하는 전역 변수로 사용됨 (clean_store_name 등에서 활용)

### 1. 함수 : crawl_reviews(restaurant_name, full_address)
```python
def crawl_reviews(restaurant_name: str, full_address: str, max_wait_sec=10) -> list[str]:
```
입력으로 음식점 이름과 전체 주소(성남시 음식점 공공 데이터에서 로드)를 받아서 네이버 지도에서 해당 음식점의 방문자 리뷰를 최대한 많이 수집합니다.

반환값은 리뷰 텍스트 리스트입니다.

### 2. 크롬 드라이버 옵션 설정 및 실행
```python
options = webdriver.ChromeOptions()
options.add_experimental_option("excludeSwitches", ["enable-logging"])
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=options)
```
로깅 생략 등 옵션을 설정하고, webdriver-manager를 통해 최신 버전의 ChromeDriver를 자동 설치 및 실행합니다.

### 3. 음식점 이름 및 주소 전처리
```python
store_name = clean_store_name(restaurant_name)
address = clean_address(full_address)
search_keyword = f"{store_name} {address}"
```

```python
def clean_store_name(name: str) -> str:
    global flag
    name = re.sub(r'\(.*?\)', '', name)  # 괄호 안 제거
    name = re.sub(r'[^가-힣\d\s]', '', name)  # 한글 + 숫자 + 공백만 허용
    name = re.sub(r'\s+', ' ', name).strip()
    if name[-1] == '점':
        flag = False
    return name
```
괄호 제거, 특수문자 제거, 의미 없는 접미사 "점" 제거(ex. OOO 분당 서현점 -> OOO 분당 서현)

마지막 글자가 "점"이면 flag = False로 설정

```python
def clean_address(address: str) -> str:
    global flag
    if not address or flag == False:
        return ""
    return re.split(r'[,(]', address)[0].strip()
```

flag == False 이면 음식점뒤에 주소를 붙이지 않음 -> 검색 실패를 방지

flag == True 라면 예를 들어 음식점 주소가 "성남시 중원구 시민로 66, 101-9호 (중앙동)" 일 경우 (중앙동)을 제외한 "성남시 중원구 시민로 66, 101-9호"를 반환



