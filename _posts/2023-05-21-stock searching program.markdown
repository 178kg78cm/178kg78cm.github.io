---
layout: post
title:  "기업 정보 크롤링"
date:   2023-05-21 19:31:29 +0900
categories: jekyll update
---

## 문제 개요
> 투자를 하기 앞서 선행되어야 하는 과정이 기업에 대한 가치 평가이다. 네이버 뉴스와 네이버 증권 사이트의 정보를 크롤링해서 나만의 기업 가치 평가 보조 프로세스를 구현한다. 

  ### install list
pip install requests
pip install beautifulsoup4
pip install pandas
pip install pillow
pip install matplotlib
pip install seaborn
pip install tkinter

## Main code 
``` python
import requests
from bs4 import BeautifulSoup
import re
import os
import time
import io
import sys
import json
from datetime import datetime, timedelta
import pandas as pd
import tkinter as tk
import webbrowser
from PIL import ImageTk, Image
import urllib.request
import matplotlib.pyplot as plt
import seaborn as sns
from matplotlib.backends.backend_agg import FigureCanvasAgg as FigureCanvas
from matplotlib.figure import Figure
from datetime import datetime, timedelta

plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['font.family'] = 'Malgun Gothic'
plt.rcParams['axes.grid'] = False

pd.set_option('display.max_columns', 250)
pd.set_option('display.max_rows', 250)
pd.set_option('display.width', 100)

pd.options.display.float_format = '{:.2f}'.format

# 데이터랩 데이터 수집 함수
def get_data(startDate, endDate, timeUnit, keywordGroups, client_id, client_secret, device=None, ages=[], gender=None):
    url = "https://openapi.naver.com/v1/datalab/search"

    # Request 본문
    body = json.dumps({
        "startDate": startDate,
        "endDate": endDate,
        "timeUnit": timeUnit,
        "keywordGroups": keywordGroups,
        "device": device,
        "ages": ages,
        "gender": gender
    }, ensure_ascii=False)

    # Response 저장 및 변환
    request = urllib.request.Request(url)
    request.add_header("X-Naver-Client-Id", client_id)
    request.add_header("X-Naver-Client-Secret", client_secret)
    request.add_header("Content-Type", "application/json")
    response = urllib.request.urlopen(request, data=body.encode("utf-8"))
    rescode = response.getcode()
    if rescode == 200:
        # Json Result
        result = json.loads(response.read())

        df = pd.DataFrame(result['results'][0]['data'])[['period']]
        for i in range(len(keywordGroups)):
            tmp = pd.DataFrame(result['results'][i]['data'])
            tmp = tmp.rename(columns={'ratio': result['results'][i]['title']})
            df = pd.merge(df, tmp, how='left', on=['period'])
        df = df.rename(columns={'period': '날짜'})
        df['날짜'] = pd.to_datetime(df['날짜'])
    else:
        print("Error Code:" + rescode)

    return df

#일일 검색어 트렌드 분석 함수
def plot_daily_trend(df):
    colList = df.columns[1:]
    n_col = len(colList)

    fig = Figure(figsize=(12, 6))
    ax = fig.add_subplot(111)
    ax.set_title('일 별 검색어 트렌드', size=10, weight='bold')
    for i in range(n_col):
        ax.plot(df['날짜'], df[colList[i]], label=colList[i])
    ax.legend(loc='upper right')

    return fig

# html soup 함수
def crawl(URL):
    response = requests.get(URL)
    html = response.text
    soup = BeautifulSoup(html, 'html.parser')
    return soup

# 키워드 분석 함수
def calculate_keyword_score(content):
    positive = {
        '매수', '흑자', '상승세', '성장', '외인', '강세',
        '긍정', '최초', '치솟', '급상승', '이익', '최고',
        '순매수', '호재', '껑충'
    }
    negative = {
        '매도', '적자', '하락세', '정체', '위축',
        '개인', '약세', '밀림', '급락', '손실', '불황',
        '순매도', '공매도', '손해', '감소', '부정', '비난'
    }
    score = 0
    for keyword in positive:
        if keyword in content:
            score += 1
    for keyword in negative:
        if keyword in content:
            score -= 1
    return score

# 데이터 추출 및 탐색 본문
def search():
    keyword = entry.get()
    # API 인증 정보 설정
    client_id = "b0OvI3MBsyvjkkSO1VzK"
    client_secret = "zqSuNm6J7e"

    # 요청 파라미터 설정
    startDate = (datetime.now() - timedelta(weeks=12)).strftime("%Y-%m-%d")
    endDate = datetime.now().strftime("%Y-%m-%d")
    timeUnit = 'week'
    device = ''
    ages = []
    gender = ''
    # 데이터 프레임 정의
    keywordGroups = [{"groupName":keyword, "keywords":[keyword]}]

    df = get_data(startDate, endDate, timeUnit, keywordGroups, client_id, client_secret, device, ages, gender)

    fig_1 = plot_daily_trend(df)

    # 차트 이미지로 변환
    canvas = FigureCanvas(fig_1)
    canvas.draw()
    image_data = io.BytesIO()
    canvas.print_png(image_data)
    image = Image.open(image_data)

    # 이미지 사이즈 조정
    image = image.resize((400, 250), Image.LANCZOS)
    photo = ImageTk.PhotoImage(image)
    image_label1.configure(image=photo)
    image_label1.image = photo

    # 기업 정보 크롤링
    URL = "https://search.naver.com/search.naver?where=nexearch&sm=top_hty&fbm=1&ie=utf8&query=" + keyword
    soup = crawl(URL)
    code_element = soup.select_one(".t_nm") # 종목코드 class
    if code_element is None:
        output_text.delete(1.0, tk.END)
        output_text.insert(tk.END, "기업 정보를 찾을 수 없습니다.") # 종목 코드 없을 때 오류 처리
        return

    code = code_element.get_text()
    URL_final = "https://finance.naver.com/item/sise.naver?code=" + code
    soup_final = crawl(URL_final)
    price = soup_final.select_one(".tah.p11").get_text()
    
    # PER ESP 동일업종평균PER 및 적정 주가 데이터
    per_table = soup_final.find('table', summary='동일업종 PER 정보')
    esp_element = soup_final.select("#tab_con1 div:nth-child(5) table tbody tr td em")[1]
    per_element = soup_final.select("#tab_con1 div:nth-child(5) table tbody tr td em")[0]

    per = per_element.get_text()
    esp = esp_element.get_text()
    per_element = per_table.select_one('td em')
    same_industry_per = per_element.get_text()
    # 적정 주가 계산
    fair_price = float(same_industry_per) * int(esp.replace(',', ''))

    # 뉴스 크롤링
    news_dict = []
    url_front = "https://finance.naver.com/item/news_news.nhn?code="
    url_last = "&page=1&sm=title_entity_id.basic&clusterId="
    url = url_front + code + url_last
    html = requests.get(url).content
    bs = BeautifulSoup(html, 'html.parser')
    all_tr = bs.findAll('td', {'class' : 'title'})
    score = 0
    for tr in all_tr:
        a_tag = tr.find('a')
        title = a_tag.get_text()
        href = "https://finance.naver.com/" + a_tag['href']

        # 뉴스 본문 크롤링
        news_response = requests.get(href)
        news_html = news_response.text
        news_soup = BeautifulSoup(news_html, 'html.parser')

        # 본문 내용 추출
        content = news_soup.select_one("#news_read").get_text()

        # 키워드 점수 계산
        score += calculate_keyword_score(content)

        # 뉴스 제목, 링크, 점수 저장
        news_dict.append((title, href, content))

    # 출력할 텍스트 생성
    price_text = f"현재 시장가 : {price}원"
    per_text = f"PER: {per}%"
    esp_text = f"ESP: {esp}원"
    fair_price_text = f"동일업종 비교 적정 주가: {fair_price}원"
    
    # 텍스트 리스트 생성
    text_list = [price_text, per_text, esp_text, fair_price_text]
    data_text.delete(1.0, tk.END)
    # 텍스트 출력
    for text in text_list:
        data_text.insert(tk.END, text + "\n")

    output_text.delete(1.0, tk.END)
    for idx, (title, href, content) in enumerate(news_dict):
        output_text.insert(tk.END, f"{idx+1}. ")
        output_text.insert(tk.END, title, "link")
        output_text.tag_bind("link", "<Button-1>", lambda event, url=href: open_link(url))
        output_text.insert(tk.END, "\n\n")
        if idx == 4:
            break

    # 차트 크롤링
    image_url = "https://ssl.pstatic.net/imgfinance/chart/item/candle/week/" + code + ".png"
    image_response = requests.get(image_url)
    image_data = image_response.content
    image = Image.open(io.BytesIO(image_data))
    image = image.resize((400, 250), Image.LANCZOS)
    photo = ImageTk.PhotoImage(image)
    image_label.configure(image=photo)
    image_label.image = photo
    
    score_text.delete(1.0, tk.END)
    score_text.insert(tk.END, f"키워드 점수: {score}")

    # news_dict를 DataFrame으로 변환
    df = pd.DataFrame(news_dict, columns=["Title", "URL", "Context"])
    # news_dict를 Excel 파일로 저장
    file_name = f"뉴스_{keyword}_{datetime.now().strftime('%Y%m%d%H%M%S')}.xlsx"
    df.to_excel(file_name, engine='openpyxl')

    # 결과 출력 후 레이어 2 이후의 레이어들을 노출
    output_layer.pack(side="top", pady=10)
    image_layer.pack(side="left", pady=10)
    data_layer.pack(side="right", pady=10)

def open_link(url):
    webbrowser.open_new(url)

# GUI 생성
window = tk.Tk()
window.title("기업 가치 평가 프로그램")
window.geometry("800x1000")
window.resizable(False, False)

# 스크롤바 생성
scrollbar = tk.Scrollbar(window)
scrollbar.pack(side=tk.RIGHT, fill=tk.Y)

# 프레임 생성
frame = tk.Frame(window)
frame.pack()

# 레이어 1 (검색 입력과 버튼)
layer1 = tk.Frame(frame)
layer1.pack(side="top", pady=10)

# 입력 상자
entry = tk.Entry(layer1, width=50)
entry.pack(side=tk.LEFT)
# 검색 버튼
search_button = tk.Button(layer1, text="검색", command=search)
search_button.pack(side=tk.LEFT)

# 레이어 2 (뉴스 출력)
output_layer = tk.Frame(frame)
output_text = tk.Text(output_layer, width=100, height=10)
output_text.pack()

# 레이어 3 (키워드 점수 출력)
score_text = tk.Text(output_layer, width=30, height=1)
score_text.pack()

# 레이어 4 (결과 출력)
data_layer = tk.Frame(frame)
# 결과 출력 텍스트 위젯
data_text = tk.Text(data_layer, width=40, height=5)
data_text.pack(side="left")

# 레이어 5 (그래프 출력)
image_layer = tk.Frame(frame)
image_label = tk.Label(image_layer)
image_label.pack()
image_label1 = tk.Label(image_layer)
image_label1.pack()

# 프로그램 실행
window.mainloop()
```

