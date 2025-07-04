# 네이버 지도 음식점 리뷰 크롤링 방법 설명
네이버 지도에서 음식점 리뷰를 크롤링 하는 과정에 대하여 상세하게 다룹니다.

## 크롤링 활동 계기
2학년 1학기 웹크롤링실습 강의 기말고사 평가 항목으로 감성분석 모델을 활용하여 웹 프로젝트를 진행하는 "성남시 음식점 리뷰 감성분석 및 분석 웹 서비스" 에 모델 학습을 위한 데이터 준비 과정으로 필요하였고,
"내돈내픽" 서비스에서 리뷰를 수집하는 과정이 필요하여 두 개의 프로젝트에 맞게 다른 목적으로 크롤링을 진행하게 되었음.

## requirements
필요한 라이브러리 : selenium, webdriver_manager (크롬 브라우저 사전에 설치되어 있어야 함)

## 진행 프로세스 별 상세 코드 설명 및 작동 결과 (CrawlerReview.py)
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

### 4. 검색어 유효성 검사
```python
if len(store_name.split()) < 1:
    print(f"❌ '{store_name}' → 검색어가 너무 짧아 스킵")
    return []
```
검색 키워드가 비정상적으로 짧으면 바로 함수 종료 -> 다음 음식점 크롤링

### 5. 네이버 지도 검색 페이지 접속 및 searchIframe 진입
```python
search_url = f"https://map.naver.com/p/search/{search_keyword}?c=15.00,0,0,0,dh"
driver.get(search_url)
time.sleep(2)
driver.switch_to.frame("searchIframe")
```


네이버 지도에 검색어로 직접 URL 접근

searchIframe으로 전환하여 검색 결과를 읽을 수 있도록 준비

### 6. 검색 결과 중 '성남' 주소를 가진 항목 탐색
```python
results = driver.find_elements(By.CSS_SELECTOR, "li.VLTHu.OW9LQ, li.UEzoS.rTjJo")
```
네이버 지도 검색 결과 리스트를 가져옴 (CSS 클래스는 페이지 구조에 따라 다름)
```python
target_element = None
        try:
            for li in results:
                try:
                    addr_el = li.find_element(By.CSS_SELECTOR, "span.lWwyx span.Pb4bU")
                    address_text = addr_el.text.strip()
                    if address_text.startswith("성남"):
                        target_element = li.find_element(By.CSS_SELECTOR, "a._T0lO")
                        print(f"✅ '{search_keyword}' → 주소 '{address_text}' 선택됨")
                        break
                except:
                    continue
        except NoSuchElementException:
            return []
```
![5-1](https://github.com/user-attachments/assets/7b8f5efa-7c85-4f31-95cb-3b40ffad178a)</br>
주소 텍스트가 "성남"으로 시작하는 항목을 찾아 클릭 대상 a 태그를 추출 -> 출발 버튼 클릭 (출발 버튼 클릭시 https://map.naver.com/p/directions/14152391.353845,4496650.856792,%EA%B7%B8%EC%88%A0%EC%A7%91,1204959745,PLACE_POI/-/-/transit?c=15.00,0,0,0,dh 와 같은 주소를 얻을 수 있는데,
이 주소를 후에 처리 예정)
```python
if target_element:
            driver.execute_script("arguments[0].click();", target_element)
            time.sleep(2)
        else:
            print(f"⚠️ '{search_keyword}' → 성남 주소 검색 결과 없음, 상세페이지로 이동 시도")
            driver.switch_to.default_content()
            time.sleep(1)
            try:
                entry_iframe = WebDriverWait(driver, 5).until(
                    EC.presence_of_element_located((By.ID, "entryIframe"))
                )
                driver.switch_to.frame(entry_iframe)
            except Exception as e:
                print(f"❌ 상세페이지 iframe 진입 실패: {search_keyword}")
                driver.quit()
                return []
```
대상 element가 존재하면 클릭하여 페이지 진입

없다면 2가지를 확인해야 함.

1. 정말로 주소가 성남으로 시작하는 음식점이 없는지 : 5초간 기다렸을 때 entryIframe으로 전환되지 않으면 return []

2. 네이버 지도에서 entryIframe을 자동으로 잡아주기 때문 -> 결과가 유일하게 하나만 존재함 : entryIframe으로 전환

### 6. URL 기반 place_id 추출
```python
try:
    current_url = driver.current_url
        if 'directions' in current_url:
            coords = re.search(r'/directions/([^/]+)', current_url).group(1).split(',')
            if len(coords) >= 4:
                place_id = coords[3]
            else:
                match = re.findall(r"place/(\d+)", current_url)
                if match:
                    place_id = match[0]
except Exception as e:
    print(f"❌ place_id 추출 실패: {search_keyword} - {e}")
    driver.quit()
    return []

if not place_id:
    print(f"❌ place_id 없음: {search_keyword}")
    driver.quit()
    return []
```
현재 URL에서 place/1234567890 형식의 숫자만 추출하여 place_id를 확보

일부 예외적으로 directions 기반 URL(출발 버튼 클릭)이면 다른 방식으로 처리 (directions/14152391.353845,4496650.856792,%EA%B7%B8%EC%88%A0%EC%A7%91,1204959745 => index가 3인 1204959745 placeId 추출)

### 7. 리뷰 탭 URL 접근 및 리뷰 수집 준비
```python
review_url = f"https://pcmap.place.naver.com/restaurant/{place_id}/review/visitor"
driver.switch_to.default_content()
driver.get(review_url)
time.sleep(2)
```

![7-1](https://github.com/user-attachments/assets/23896803-a9a0-4097-84be-31657852b66b)

place_id를 바탕으로 리뷰 전용 URL을 직접 생성

iframe에서 벗어나 새 URL 접근

### 8. "더보기" 버튼 최대 20회 클릭 (리뷰 더 가져오기)
```python
more_click_count = 0
    while more_click_count < 20:
        try:
            more_btn = driver.find_element(By.CSS_SELECTOR, '.lfH3O > a.fvwqf')
            more_btn.click()
            more_click_count += 1
            print(f"더보기 {more_click_count} 클릭")
            time.sleep(1)
        except:
            break
```

![8-1](https://github.com/user-attachments/assets/5b2a07c4-9b81-43ee-a5dd-f2354c71538d)

리뷰 더보기 버튼이 존재할 경우 계속 클릭 (최대 20회까지)

한 번 클릭 시마다 5~10개 정도 리뷰가 추가 로딩됨

### 9. 리뷰 텍스트 추출 후 드라이버 종료, return reiviews
```python
# (8) 리뷰 추출
    review_items = driver.find_elements(By.CSS_SELECTOR, 'li.place_apply_pui.EjjAW')
    print(f"[{store_name}] 총 수집 리뷰 개수: {len(review_items)}개")

    for i, li in enumerate(review_items, start=1):
        try:
            review_text = li.find_element(By.CSS_SELECTOR, 'div.pui__vn15t2 a').text.strip()
            reviews.append(review_text)
        except:
            continue

finally:
    driver.quit()

return reviews
```
각 리뷰 요소에서 텍스트를 추출하여 reviews 리스트에 저장

실패 시 try-except로 무시하고 다음 항목으로 진행

### 10. 최종 반환 예시
```python
[
  "친절하고 양도 많아요. 다시 오고 싶어요!",
  "맛은 그냥 그랬는데 대기가 너무 길었어요...",
  "가성비 좋고 위치도 괜찮아요~"
]
```

## 크롤링한 리뷰 활용
["음식점 리뷰 감성분석 모델 학습 과정"](https://github.com/tpdus751/restaurant_review_sentiment_analysis_model)의 사전 준비로 리뷰 감성 분석 학습용 입력 데이터 구축을 위한 라벨링 진행과 연결됨.
