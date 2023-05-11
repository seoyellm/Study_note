# GIS 시각화 프로젝트 - VISIT JEJU

## 1. 데이터 불러오기

​																						-	00.data_collect.ipynb



## 2. flask_run_file

### 2.1 기본 작업, import

``` python
#------------------------------------------------
# pip install Flask,  requests
# ----------------------------------OpenAPI KEY : cptjvrh7ki6dxd9q
# 제주관광공사_비짓제주 관광정보 오픈 (API)
# https://www.data.go.kr/data/15076361/openapi.do
# http://api.visitjeju.net/vsjApi/contents/searchlist
#------------------------------------------------

from flask import Flask, session, render_template, make_response, jsonify, request, redirect, url_for

import cx_Oracle
import pandas as pd
import numpy as np
import random

import json
import folium
from folium import plugins
import re
import googlemaps
import pprint
import requests

app = Flask(__name__)
app.secret_key = "1111122222"


#########################################################################################################
"""
CREATE TABLE JEJU_USERS 
(
  SEQ VARCHAR2(20) PRIMARY KEY 
, USERID VARCHAR2(20) NOT NULL 
, COLUMN2 VARCHAR2(20) NOT NULL 
, USERNM VARCHAR2(20) NOT NULL 
);
create sequence JEJU_USERS_SEQ start with 1 increment by 1;
insert into jeju_users values(JEJU_USERS_SEQ.nextval, 'lee', '111','아무개');
insert into jeju_users values(JEJU_USERS_SEQ.nextval, 'hong', '222','홍길동');
commit;
"""

```

### 2.1 login 화면

``` python
""" -----------------------------------------------------
[화면] 로그인 폼
 ----------------------------------------------------- """
@app.route('/login_form', methods=["get"])
def login_form():
    return render_template('login_form.html')

""" -----------------------------------------------------
[처리부] 로그인 
 ----------------------------------------------------- """
@app.route('/login', methods=["post"])
def login():
    userid = request.form.get("userid")
    userpw = request.form.get("userpw")

    session['USER_ID_SESSION'] = userid
    return redirect("/")

""" -----------------------------------------------------
[처리부] 로그아웃
 ----------------------------------------------------- """
@app.route('/logout', methods=["get"])
def logout():
    session.pop('USER_ID_SESSION')
    return redirect("/")

""" -----------------------------------------------------
[화면] 회원가입 폼
 ----------------------------------------------------- """
@app.route('/register_form', methods=["get"])
def register_form():
    return render_template('register_form.html')
```

### 2.3 공통함수

``` python
""" -----------------------------------------------------
[공통함수] 리스트+지도마커 생성
 ----------------------------------------------------- """
def make_map_html(cate='숙박', search_str="", contentsid_list=None):
    df = pd.read_csv("./datasets_jeju/data.csv")
    print(df['label'].value_counts())

    """ -----------------------------------------------------
    [처리부] 리스트 
    ----------------------------------------------------- """
    #----------- 메인화면 : 관광명소 검색 시
    if len(search_str) > 0 :
        df = df[ df['title'].str.contains(search_str) ]
    # ----------- 로그인 > 마이페이지 : 찜목록 출력 시
    elif contentsid_list is not None:
        df = df[df['contentsid'].isin( contentsid_list ) ]   # ['CONT_000000000500349', 'CONT_000000000500477', 'CNTS_000000000001195']
    # ----------- 메인화면 기본 관광지 출력
    else :
        df = df[ df['label'] == cate ]

    #df['tag'] = df['tag'].apply(lambda x: x.split(",")[0] if len(x) > 0 else cate)
    df = df[['contentsid', 'title', 'latitude','longitude', 'address', 'phoneno', 'tag', 'likecnt', 'reviewcnt', 'evelpt', 'thumb']]


    """ -----------------------------------------------------
    [처리부] 지도마커 
    ----------------------------------------------------- """
    geo_list  = []
    name_list = []
    for i in range(len(df)):
        lat = df.iloc[i]['latitude']
        lng = df.iloc[i]['longitude']
        sname = df.iloc[i]['title']
        sid = df.iloc[i]['contentsid']
        geo_list.append((lat, lng))
        popup_url = f"'/detail_view?contentsid={sid}'"
        popup_option = "'status=no,menubar=no,toolbar=no,scrollbars=yes,resizable=yes,width=400,height=600'"
        popup_html = f"""<font size=2><b><a href=\"javascript:window.open({popup_url},'',{popup_option})\">{sname}</a></b></font><br> 
                        {df.iloc[i]['address']}
                        <img src='{df.iloc[i]['thumb']}' width=150 height=100>
                        {df.iloc[i]['phoneno']}"""
        name_list.append(popup_html)
    map = folium.Map(location=[33.41041350000001, 126.4913534], zoom_start=11,
                     tiles='OpenStreetMap')  # Stamen Terrain')
    plugins.MarkerCluster(geo_list, popups=name_list).add_to(map)
    # ---------------------------------------------------
    # web browser에 보이기 위한 준비
    map.get_root().width = "800px"
    map.get_root().height = "600px"
    html_str = map.get_root()._repr_html_()

    return html_str, df[:4]
```

### 2.4 처리부(RESTful)

``` python
""" -----------------------------------------------------
[처리부 : REST] 메인화면 우측 지도하단 네이게시션 바 - 카테별 리스트+지도마커 생성
        카테 : '음식점', '관광지', '숙박', '쇼핑' 
 ----------------------------------------------------- """
@app.route('/search_map_html') #, methods=['''POST'])
def search_map_html(cate='숙박', search_str=""):
    # prm_dic = request.get_json()
    # print(prm_dic['cate'])

    if request.method == "GET":
        cate = request.args.get("cate")
        search_str = request.args.get("search_str")
        print("GET", cate, search_str)
    elif request.method == "POST":
        cate = request.form.get("cate", default='숙박')
        search_str = request.form.get("search_str", default='숙박')
        print("POST", cate, search_str)

    if bool(cate) == False:
        cate = '숙박'
    if bool(search_str) == False:
        search_str = ''

    """ -----------------------------------------------------
    [호출] 리스트+지도마커 공통함수 
    ----------------------------------------------------- """
    html_str, df = make_map_html(cate=cate, search_str=search_str)

    #-------------------STRING으로 전달할 경우 ----------------------
    # df_json_str = df.to_json(orient='records')
    # res_string = json.dumps({"KEY_MAP_HTML": html_str, "KEY_DF_JSON": df_json_str})
    # return res_string

    # -------------------OBJECT로 전달할 경우 ----------------------
    df_json_str = df.to_json(orient='records')
    res_object  = {"KEY_MAP_HTML": html_str, "KEY_DF_JSON_STR": df_json_str}
    #print(type(jsonify( res_object )), jsonify( res_object ))   # <Response 311603 bytes [200 OK]>
    #print(type(json.dumps(res_object)) ,  json.dumps(res_object) )
    #return jsonify( res_object )
    return json.dumps(res_object)


""" -----------------------------------------------------
[화면] 메인
 ----------------------------------------------------- """
@app.route('/', methods=["get", "post"])
def index():
    # session['USER_ID_SESSION'] = USER_ID
    if request.method == "GET":
        cate = request.args.get("cate")
        search_str = request.args.get("search_str")
        print("GET", cate, search_str)
    elif request.method == "POST":
        cate = request.form.get("cate", default='숙박')
        search_str = request.form.get("search_str", default='숙박')
        print("POST", cate, search_str)


    """ -----------------------------------------------------
    [호출] 리스트+지도마커 공통함수 
    ----------------------------------------------------- """
    if bool(cate) == False :
        cate = '숙박'
    if bool(search_str) == False:
        search_str = ''
    html_str, df = make_map_html(cate=cate, search_str=search_str)
    df_json_str = df.to_json(orient='records')
    df_json     = json.loads(df_json_str)

    return render_template('index.html' ,
                           KEY_CATE=cate,
                           KEY_MAP_HTML=html_str,
                           KEY_DF_JSON_STR=df_json)


```

### 2.5 처리부(화면)

``` python
""" -----------------------------------------------------
[처리부 -> 화면 : 찜목록] 리스트+지도마커 생성 --> 화면 이동
 ----------------------------------------------------- """
@app.route('/booking', methods=["get"])
def booking():
    if session['USER_ID_SESSION'] == False:
        return redirect("/login_form")
    else :
        try :
            """ -----------------------------------------------------
           [호출] 리스트+지도마커 공통함수 
           ----------------------------------------------------- """
            with open(file=f"./user_booking/{session['USER_ID_SESSION']}.txt", encoding='UTF-8', mode='r') as fr:
                contentsid_list = fr.readlines()
            contentsid_list = [contentsid.strip()  for contentsid in contentsid_list]
            html_str, df = make_map_html(cate="", search_str="", contentsid_list= contentsid_list)
            df_json_str = df.to_json(orient='records')
            df_json = json.loads(df_json_str)

            return render_template('booking.html',
                                   KEY_MAP_HTML=html_str,
                                   KEY_DF_JSON=df_json)

        except :
            return redirect("/")


@app.route('/booking_add', methods=["get"])
def booking_add():
    contentsid = request.args.get("contentsid")
    print("GET", contentsid)

    if bool(contentsid) == False:
        return "500"
    elif session['USER_ID_SESSION'] == False:
        return "403"
    else :
        try :
            with open(file=f"./user_booking/{session['USER_ID_SESSION']}.txt", encoding='UTF-8', mode='a') as fw:
                fw.write(contentsid+"\n")
            return "200"
        except :
            return "500"
```

### 2.6 처리부(상세화면)

``` python
""" -----------------------------------------------------
[처리부 : 상세화면] 관련 유튜브영상 가져오기
# pip install youtube-search-python
 ----------------------------------------------------- """
from youtubesearchpython import VideosSearch
# import pandas as pd
# import json
#------------------------------------------------------------
def my_youtube_search(search_str='제주도', nrows=3) :
	videosSearch = VideosSearch(search_str, limit=nrows)
	json_res = videosSearch.result()
	#print(videosSearch.result())  ## [{},{},{}]
	#print(json.dumps(videosSearch.result(), sort_keys=True, indent=4))
	movie_list = json_res['result']
	#print(json.dumps(movie_list, sort_keys=True, indent=4))

	tot_list = []
	for movie in movie_list:
		dict = {}
		# print(movie['thumbnails'][0]['url'])
		# print(movie['link'])
		# print(movie['title'])
		dict["title"] 	 = movie['title']
		try :
			dict["movie"] = movie['richThumbnail']['url']
		except :
			dict["movie"] = movie['thumbnails'][0]['url']

		dict["img"] 	 = movie['thumbnails'][0]['url']
		dict["duration"] = movie['duration']
		dict["url"] 	 = movie['link']
		dict["rdate"] 	 = movie['publishedTime']
		dict["cnt"]  = movie['viewCount']['text']
		tot_list.append(dict)
	return tot_list  #[{},{}]


""" -----------------------------------------------------
[처리부 : 상세화면] 관련 리뷰댓글 가져오기
# pip install youtube-search-python
 ----------------------------------------------------- """
def get_review_list(contentsid) :
    contentsid = "CONT_000000000500811"
    headers_val={
    'Cookie':'_gcl_au=1.1.1634613698.1677463189; Hm_lvt_f6d5e380b023c5b1e946de764e07a2a2=1677463189; _fbp=fb.1.1677463189486.1921927163; iceJWT=SDP+eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJBbm9ueW1vdXMiLCJhdWQiOiJlMjQ0NGQwOC05YjA0LTRjYmEtYWE5ZS1mZjdmYWQwNTFiZWIiLCJqdGkiOiJlMjQ0NGQwOC05YjA0LTRjYmEtYWE5ZS1mZjdmYWQwNTFiZWIiLCJpc3MiOiJJLU9OIiwiaWF0IjoxNjc3NzI5NTQwLCJleHAiOjE2Nzc4Mzc1NDB9.1vOyTpGacBX1tpd0HWvdiyIZJqwVSOgeg1LjzUsyLLOQKSe97czp7VAzOTi-_22JGGWRp1pBpdydYbvnGc1Svg; iceRefreshJWT=SDP+eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJBbm9ueW1vdXMiLCJhdWQiOiJiMjJiZDkyZC1lMDc5LTRhZTgtOGFhMy05NWQwZTJkNjY1OTMiLCJpc3MiOiJJLU9OIiwianRpIjoiZjY0NGE4NzMtZGNlMi00YTQ1LTljN2MtMThkZWNjMmVmYTcwIiwiaWF0IjoxNjc3NzI5NTQwLCJleHAiOjQxMDI0MTI0MDB9.JksFE8AaH3_BjU8YBiYy7KftXVYicOgc4coi4LZeJ_OsDButjABpsXuGN8qE6kpmgH1hh-KijjZnRdbiSQJZJg; _gid=GA1.2.1598970783.1677729565; Hm_lpvt_f6d5e380b023c5b1e946de764e07a2a2=1677730295; _ga=GA1.1.1913362389.1677463189; _ga_WR7WEFWP2T=GS1.1.1677729564.5.1.1677730541.0.0.0',
    'Host': 'api.visitjeju.net',
    }
    res = requests.get(f"https://api.visitjeju.net/api/contentsreview/list.json?_siteId=jejuavj&locale=kr&device=pc&contentsid={contentsid}&noUserid=xxxxx&sorting=created+desc,created+desc&pageSize=5&page=1"
                      , headers=headers_val)
    json_dic = json.loads(res.text)
    # print( json_dic )
    # print( len(json_dic["items"]) , json_dic["items"] )
    review_list = []
    for dic in json_dic["items"]:
        if bool(dic["created"]):
            created = dic["created"][:4] + "-" + dic["created"][4:6] + "-" + dic["created"][6:8]
        # print( dic["evelpt"], dic["tag"], dic["evelcontent"], dic["created"], dic["userid"]["usernm"])
        review_list.append( {"evelpt" : dic["evelpt"],
                             "tag" : dic["tag"],
                             "evelcontent" : dic["evelcontent"],
                             "created" : created,
                             "usernm" : dic["userid"]["usernm"]
                           })
    return review_list



""" -----------------------------------------------------
[화면] 상세화면 
 ----------------------------------------------------- """
@app.route('/detail_view', methods=['GET','POST'])
def detail():
    if request.method == "GET" :
        contentsid = request.args.get("contentsid", default='CONT_000000000500811')
        print("GET", contentsid)
    elif request.method == "POST" :
        contentsid = request.form.get("contentsid", default='CONT_000000000500349')
        print("POST", contentsid)
    else:
        contentsid = 'CONT_000000000500349'
        print("etc", contentsid)

    """ -----------------------------------------------------
   [호출] 리스트+지도마커 공통함수 
   ----------------------------------------------------- """
    html_str, df = make_map_html(cate="", search_str="",contentsid_list=[contentsid])
    df_json_str = df.to_json(orient='records')
    df_json = json.loads(df_json_str)


    """ -----------------------------------------------------
    [호출] 관련댓글
    ----------------------------------------------------- """
    try :
        review_list = get_review_list(contentsid)
        print(review_list)
    except :
        review_list = []

    """ -----------------------------------------------------
    [호출] 유튜브영상 
    ----------------------------------------------------- """
    try:
        search_str = df['title'].values[0]
        if bool(search_str) == False:
            search_str = "제주도"
        print(search_str)
        movie_list = my_youtube_search(search_str, 4)  # [  {title:}, {title:}, {}]
        print(movie_list)
    except:
        movie_list = []

    return render_template('detail.html', KEY_MAP_HTML=html_str
                           , KEY_DF=df_json[0]
                           , KEY_YT_LIST=movie_list
                           , KEY_REVIEW_LIST=review_list
                           )
```

### 2.7 chart

``` python
@app.route('/chart')
def chart():
    return render_template('chart_test.html')


if __name__ == '__main__':
    app.debug = True
    app.run(host='0.0.0.0', port=8877)
```





## 2. index.html

### 2.1 login

``` html
   <!-- header section start -->
    <header class="header-section">
        <!-- top bar -->
        <div class="top-bar-header">
            <div class="container">
                <div class="row">
                    <div class="col-xl-6 col-md-8 col-12">
                        <div class="top-bar-left">
                            <span>Braking offer: 50% Offer  of our aparment for 1 month</span>
                        </div>
                    </div>
                    <div class="col-xl-6 col-md-4 col-12 text-md-right">
                        <div class="top-bar-right">
                            <div class="user-setting">
                                 <ul>
{% if session['USER_ID_SESSION'] %}
    <li><a href="/logout"><i class="fas fa-sign-out-alt"></i>Logout</a></li>
{% else %}
    <li><a href="/login_form"><i class="fas fa-sign-out-alt"></i>Login</a></li>
    <li><a href="/register_form"><i class="fas fa-user"></i>Register</a></li>
{% endif %}
                                </ul>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <!-- main menu -->
        <div class="main-header">
            <div class="container">
                <div class="row">
                    <div class="col-md-3 logo">
                        <a class="navbar-brand" href="index.html">
                            <img src="static/assets/img/logo.png" class="logo-display" alt="shipo">
                        </a> 
                    </div> <!-- /.col-md-3 logo -->
                    <div class="col-md-9 d-none d-lg-block text-lg-right">
                        <nav id="responsive-menu" class="main-menu">
                            <ul class="menu-items">
                                <li><a href="#">home</a>
                                    <ul class="submenu">
                                        <li><a href="index.html">Home One</a></li>
                                        <li><a href="index-2.html">Home Two</a></li>
                                    </ul>    
                                </li>
                                 <li><a href="about.html">about</a></li>
                                 <li><a href="listing-single-details.html">Listing</a>
                                    <ul class="submenu">
                                        <li><a href="tags.html">tags</a></li>
                                        <li><a href="tags-grid.html">tags grid</a></li>
                                        <li><a href="explors-map-sidebar.html">Explors</a></li>
                                    </ul>    
                                 </li>
                                 <li><a href="explors-map-sidebar.html">Property</a></li>
                                 <li><a href="explors-map-sidebar.html">Find</a></li>
                                 <li><a href="contact.html">Contact</a></li>
                                <li><a href="#">Pages</a>
                                    <ul class="submenu">
                                        <li><a href="login.html">login</a></li>
                                        <li><a href="register.html">register</a></li>
                                        <li><a href="booking.html">booking</a></li>
                                        <li><a href="popular-locations.html">Popular Place</a></li>
                                        <li><a href="contact.html">contact</a></li>
                                    </ul> 
                                </li>
                            </ul>
{% if session['USER_ID_SESSION'] %}
        <a href="/booking" class="btn-booking-start">Booking Now</a>
 {% endif %}    <div class="col-xl-6 col-md-4 col-12 text-md-right">
        <div class="top-bar-right">
            <div class="user-setting">
                 <ul>
{% if session['USER_ID_SESSION'] %}
    <li><a href="/logout"><i class="fas fa-sign-out-alt"></i>Logout</a></li>
{% else %}
    <li><a href="/login_form"><i class="fas fa-sign-out-alt"></i>Login</a></li>
    <li><a href="/register_form"><i class="fas fa-user"></i>Register</a></li>
{% endif %}
                </ul>
            </div>
        </div>
    </div>
```

### 2.2 검색창

``` html
<!-- 검색창 ====================================================================================== -->
                <div class="search-section d-flex col-lg-2 col-sm-12 p-0 bg-cover" style="background-image: url('static/assets/img/search-bg.jpg')">
                    <div class="search-tab-wrap">
                        <!-- Nav tabs -->
                        <ul class="nav nav-tabs">
                           <li class="nav-item">
                              <a class="nav-link active" data-toggle="tab" href="#place">place</a>
                           </li>
                           <li class="nav-item">
                              <a class="nav-link" data-toggle="tab" href="#event">event</a>
                           </li>
                        </ul>
                        <!-- Tab panes -->
                        <div class="tab-content">
                           <div class="tab-pane active container" id="place">
                               <div class="search-form-box">
                                   <form action="/" method="post" class="search-form">
                                       <input type="text" placeholder="검색어" name="search_str">
                                       <select name="cate">
                                           <option value="전체">전체</option>
                                           <option value="음식점">음식점</option>
                                           <option value="관광지">관광지</option>
                                           <option value="숙박">숙박</option>
                                           <option value="쇼핑">쇼핑</option>
                                       </select>
                                       <button type="submit">장소 검색</button>
                                   </form>
                               </div>
                           </div>
                           <div class="tab-pane container" id="event">
                                <div class="search-form-box">
                                   <h3>Discover Great Event.</h3>
                                   <p>Tag: Nightlife, Hotel, Restaurant, Sport, Shop.</p>
                                   <form action="#" class="search-form">
                                       <input type="text" placeholder="Where to go?" name="place">
                                       <input type="text" placeholder="Location" name="location">
                                       <select>
                                           <option value="Categroy">Categroy</option>
                                           <option value="1">Hotel</option>
                                           <option value="2">Restaurant</option>
                                           <option value="3">Property</option>
                                           <option value="4">Shopping</option>
                                       </select>
                                       <button type="submit">Search request</button>
                                   </form>
                               </div>
                           </div>
                        </div>
                    </div>
                </div>
```

### 2.3 카드맵

``` html
<!-- 카드맵 ====================================================================================== -->
                <div class="col-lg-5 col-sm-12">

                    <div class="row" id="my-top-n-div">
{% for dic in KEY_DF_JSON_STR %}
                        <div class="col-xl-6 col-md-6 col-lg-6 col-sm-12">
                            <div class="single-popular-item">
<div class="item-cover-image bg-cover" style="background-image: url('{{dic['thumb'] | safe}}')">
                                    <div class="tags">
                                        <ul>
<li><a href="#">{{dic['tag']}}</a></li>
                                        </ul>
<a href='#' class='btn-wishlist' value="{{dic['contentsid']}}"><i class="far fa-heart"></i></a>
                                    </div>
                                    <div class="item-details">
                                        <div class="item-category">
                                            <div class="cat-icon">
                                                <img src="static/assets/img/exterior.png" alt="listico">
                                            </div>
                                        </div>
                                    </div>
                                </div>
                                <div class="item-content">
                                    <h4><a href=javascript:window.open("/detail_view?contentsid={{dic['contentsid']}}",'',"status=no,menubar=no,toolbar=no,scrollbars=yes,resizable=yes,width=400,height=600")>
{{dic['title']}}</a></h4>
                                    <div class="item-feedback">
                                        <ul>
{% for i in range( (dic['evelpt'] | int )) %}
                                                <li><i class="fas fa-star"></i></li>
{% endfor %}
                                        </ul>
({{dic['reviewcnt']}} Reviwer)
                                    </div>
                                    <div class="item-store-location">
                                        <i class="fas fa-map-marker-alt"></i>
{{dic['address']}}
                                    </div>

                                </div>
                            </div>
                        </div>
{% endfor %}
                    </div>


                </div>
```

### 2.4 지도

```html
<!-- 지도  -------------------------------------------------------------------------------  -->
                    <div class="slides listico-slider">
                        <div class="single-slide" id="div_travel">
                            {{ KEY_MAP_HTML|safe if KEY_MAP_HTML is not none }}
                        </div>
                       <div class="single-slide">
                            {{ KEY_MAP_HTML|safe if KEY_MAP_HTML is not none }}
                        </div>
                        <div class="single-slide">
                            {{ KEY_MAP_HTML|safe if KEY_MAP_HTML is not none }}
                        </div>
                        <div class="single-slide">
                            {{ KEY_MAP_HTML|safe if KEY_MAP_HTML is not none }}
                        </div>
                        <div class="single-slide">
                            {{ KEY_MAP_HTML|safe if KEY_MAP_HTML is not none }}
                        </div>
                    </div>
<!-- 지도 네비 ---------------------------------------------------------------------------- -->
                    <div class="slides slider-nav text-center">
<div class="single-nav {%if KEY_CATE=='쇼핑'%} slick-current {%endif%}" {%if KEY_CATE=='쇼핑'%} tabindex="0" {%endif%} value="쇼핑">
                            <div class="slide-icon">
                                <span>쇼핑</span>
                                <img src="static/assets/img/shop.png" alt="listico">
                                2 Listing
                            </div>
                        </div>
<div class="single-nav {%if KEY_CATE=='음식점'%} slick-current {%endif%}" {%if KEY_CATE=='음식점'%} tabindex="0" {%endif%} value="음식점">
                            <div class="slide-icon" >
                                <span>음식점</span>
                                <img src="static/assets/img/red-exterior.png" alt="listico">
                                8 Listing
                            </div>
                        </div>


<div class="single-nav {%if KEY_CATE=='숙박'%} slick-current {%endif%}" {%if KEY_CATE=='숙박'%} tabindex="0" {%endif%} value="숙박">
                            <div class="slide-icon" >
                                <span>숙박</span>
                                <img src="static/assets/img/red-kat.png" alt="listico">
                                2 Listing
                            </div>
                        </div>
<div class="single-nav {%if KEY_CATE=='관광지'%} slick-current {%endif%}" {%if KEY_CATE=='관광지'%} tabindex="0" {%endif%} value="관광지">
                            <div class="slide-icon" >
                                <span>관광지</span>
                                <img src="static/assets/img/red-din.png" alt="listico">
                                8 Listing
                            </div>
                        </div>
<div class="single-nav  {%if KEY_CATE=='숙박'%} slick-current {%endif%}" {%if KEY_CATE=='숙박'%} tabindex="0" {%endif%} value="숙박">
                            <div class="slide-icon" >
                                <span>숙박</span>
                                <img src="static/assets/img/red-medical.png" alt="listico">
                                5 Listing
                            </div>
                        </div>

                    </div>
                </div>
            </div>
       </div>
    </section>

```

### 2.5 AJAX(js)

``` js
<script>




$(document).ready(function(){


    $(document).on("click",".btn-wishlist",function() {
        alert($(this).attr("value"))
        $.ajax({
			  url: "/booking_add" ,
			  data: {"contentsid":$(this).attr("value")} ,
			  contentType : "application/x-www-form-urlencoded; charset=UTF-8",
			  dataType    : "text" ,
              success: function( resString ) {           //콜백함수처리 방식 : JSON 객체로 들어온다
                    console.log(resString)               //200
              },
              error: function( result ) {
                    alert("error");                     //403, 500
              },
        });
    });


    $(document).on("click",".single-nav",function() {

        //------ /rest 비동기통신 --------------------------------
		$.ajax({
			  url: "./search_map_html" ,
			 //----------------------------------------------------------------------
			  data: {"cate":$(this).attr("value")} ,
			  contentType : "application/x-www-form-urlencoded; charset=UTF-8",
			 //----------------------------------------------------------------------
			 //data        : JSON.stringify({"cate":$(this).val()}) ,
             //contentType : "application/json; charset=UTF-8",
             //=======================================================================
			 //dataType    : "text" ,
             //success: function( resString ) {           //콜백함수처리 방식 : 문자열로 들어온다
             //     console.log(resString)
             ///    resObject = JSON.parse(resString);   //{k1:val, k2:val2}
             //----------------------------------------------------------------------
              dataType    : "json" ,
              success: function( resObject ) {           //콜백함수처리 방식 : JSON 객체로 들어온다
                    console.log(resObject)               //{k1:val, k2:val2}
             //----------------------------------------------------------------------
             // { KEY_MAP_HTML    : '<iframe srcdoc="&lt;!DOCTYPE html&gt;\n&lt;html...</iframe>',
             //   KEY_DF_JSON_STR : '[{"title":"\\ub098\\ubbf8\\", ..., "img":"3-9914-e485793e3b5a.jpg"}]'
             //  }

                    var key_map_html = resObject["KEY_MAP_HTML"];
                    var key_df_json  = JSON.parse(resObject["KEY_DF_JSON_STR"]);
                    console.log(key_map_html)
                    console.log(key_df_json)             //{k1:val, k2:val2}
                    htmlstr = "";

					$.map( key_df_json, function( val, idx ) {
					    console.log( val['thumb'] )

					    htmlstr += "<div class='col-xl-6 col-md-6 col-lg-6 col-sm-12'>"
                        htmlstr += "<div class='single-popular-item'>"
                        htmlstr += "<div class='item-cover-image bg-cover' style=\"background-image: url('"+val["thumb"]+"')\">"
                        htmlstr += "<div class='tags'>"
                        htmlstr += "<ul><li><a href='#'>" + val['tag'] + "</a></li></ul>"
                        htmlstr += "<a href='#' class='btn-wishlist' value='" + val['contentsid'] + "'><i class='far fa-heart'></i></a>"
                        htmlstr += "</div>"
                        htmlstr += "<div class='item-details'><div class='item-category'><div class='cat-icon'>"
                        htmlstr += "<img src='static/assets/img/exterior.png' alt='listico'>"
                        htmlstr += "</div></div></div>"
                        htmlstr += "</div>"
                        htmlstr += "<div class='item-content'>"
                        htmlstr += "<h4><a href='/detail_view?contentsid="+val['contentsid']+"'>" + val['title'] + "</a></h4>"
                        htmlstr += "<div class='item-feedback'>"
                        htmlstr += "<ul>"

                        var evelpt = parseInt(val['evelpt']);
                        for (var i = 0; i < evelpt; i++) {
                            htmlstr += "<li><i class='fas fa-star'></i></li>"
                        }

                        htmlstr += "</ul>"
                        htmlstr += "("+val["reviewcnt"]+" Reviwer)"
                        htmlstr += "</div>"
                        htmlstr += "<div class='item-store-location'>"
                        htmlstr += "<i class='fas fa-map-marker-alt'></i>"
                        htmlstr += val["address"]
                        htmlstr += "</div></div></div></div>"
					});

                    $("#my-top-n-div").empty();
                    $("#my-top-n-div").html(htmlstr);

					$(".single-slide").empty();
                    $(".single-slide").html(key_map_html);

			  },
			  error: function( result ) {
					alert("error");
			  },
		});
   });


 });
</script>
```





## 3. booking.html

### 3.1  top bar

``` html
<!-- top bar -->
        <div class="top-bar-header">
            <div class="container">
                <div class="row">
                    <div class="col-xl-6 col-md-8 col-12">
                        <div class="top-bar-left">
                            <span>Braking offer: 50% Offer  of our aparment for 1 month</span>
                        </div>
                    </div>
                    <div class="col-xl-6 col-md-4 col-12 text-md-right">
                        <div class="top-bar-right">
                            <div class="user-setting">
                                 <ul>
{% if session['USER_ID_SESSION'] %}
    <li><a href="/logout"><i class="fas fa-sign-out-alt"></i>Logout</a></li>
{% else %}
	<li><a href="/login_form"><i class="fas fa-sign-out-alt"></i>Login</a></li>
	<li><a href="/register_form"><i class="fas fa-user"></i>Register</a></li>
{% endif %}
                                </ul>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
```

### 3.2 main menu

``` html
{% if session['USER_ID_SESSION'] %}
        <a href="/booking" class="btn-booking-start">Booking Now</a>
 {% endif %}
```

### 3.3 상단지도

```html
    <div class="banner-section">
        <div class="gmap-banner">
           {{ KEY_MAP_HTML|safe if KEY_MAP_HTML is not none }}
        </div>
    </div>
```



### 3.4 하단 즐겨찾기 장소

```html
<section class="tag-item-section section-padding page-bg">
        <div class="container">
            <div class="row">
{% for dic in KEY_DF_JSON %}
                    <div class="col-md-4 col-sm-12">
                    <div class="single-popular-item">
<div class="item-cover-image bg-cover" style="background-image: url('{{dic['thumb'] | safe}}')">
                            <div class="tags">
                                <ul>
<li><a href="#">{{dic['tag']}}</a></li>
                                </ul>
                                <a href="#" class="btn-wishlist"><i class="far fa-heart"></i></a>
                            </div>
                            <div class="item-details">
                                <div class="item-category">
                                    <div class="cat-icon">
                                        <img src="static/assets/img/exterior.png" alt="listico">
                                    </div>
                                </div>
                            </div>
                        </div>
                        <div class="item-content">
<h4><a href=javascript:window.open("/detail_view?contentsid={{dic['contentsid']}}",'',"status=no,menubar=no,toolbar=no,scrollbars=yes,resizable=yes,width=400,height=600")>
{{dic['title']}}</a></h4>
                            <div class="item-feedback">
{% for i in range( (dic['evelpt'] | int )) %}
	<li><i class="fas fa-star"></i></li>
{% endfor %}
({{dic['reviewcnt']}} Reviwer)
                            </div>
<div class="item-store-location">
    <i class="fas fa-map-marker-alt"></i>
     {{dic['address']}}
</div>

                        </div>
                    </div>
                </div>
{% endfor %}

            </div>
            <div class="load-ajx text-center">
                <a href="#" class="btn-loadmore">Load More</a>
            </div>
        </div>
    </section>
```





## 4. detail.html

### 4.1 single deatils content wrapper start

``` html
<section class="single-details-box-section section-padding page-bg">
        <div class="container">
            <div class="row">
                <div class="col-lg-12 col-sm-12">
                    <div class="single-content-box-area">
                        <div class="description_box box" id="des-box">
<h3>{{KEY_DF['title']}}</h3><a href='#' class='btn-wishlist' value="{{KEY_DF['contentsid']}}"><i class="far fa-heart"></i></a>
<p>{{KEY_DF['sbst']}}</p>
                        </div>

<!----------------- 포토 슬라이싱 ------------------>
                        <div class="photos_box box" id="photo-box">
                            <h3>관련사진</h3>
                            <div class="photo-gallery">
<div class="single-g-slide">
    <img src="{{KEY_DF['img']}}" alt="">
</div>
                                <div class="single-g-slide">
                                    <img src="static/assets/img/gallery/gallery3.jpg" alt="">
                                </div>
<div class="single-g-slide">
    <img src="{{KEY_DF['thumb']}}" alt="">
</div>
                                <div class="single-g-slide">
                                    <img src="static/assets/img/gallery/gallery4.jpg" alt="">
                                </div>
                            </div>
                            <div class="gallery_nav">
<div class="single-gallery-img">
	<img src="{{KEY_DF['img']}}" alt="">
</div>
                                <div class="single-gallery-img">
                                    <img src="static/assets/img/gallery/gn3.jpg" alt="">
                                </div>
<div class="single-gallery-img">
    <img src="{{KEY_DF['thumb']}}" alt="">
</div>
                                <div class="single-gallery-img">
                                    <img src="static/assets/img/gallery/gn4.jpg" alt="">
                                </div>
                            </div>
                        </div>
```

### 4.2 지도

``` html
                        <div class="map_box box" id="map-box">
                            <h3>지도위치</h3>
                            <div class="map-wrap">
                                {{KEY_MAP_HTML | safe}}<br>
                                {{KEY_DF['address']}}<br>
                                {{KEY_DF['phoneno']}}
                            </div>
                        </div>
```

### 4.3 동영상

```html
                        <div class="video_box box" id="video-box">
                            <h3>관련영상</h3>
                            <div class="video-wrap">

                                {% for dict in KEY_YT_LIST %}
                                    {{dict['title']}}<br>
                                    <a href="movies/suicide-squad/index.html">
                                        <img src="{{dict['movie']}}" width="480" height="400"/><br>
                                    </a>
                                {% endfor %}

                            </div>
                        </div>
```

### 4.4 관련글

```html
 					<div class="menu_price box" id="menu-price">
                            <h3>관련댓글</h3>
                            <div class="menu-item-price">
                                    <div class="item-feedback">
                                        <hr>
                                            {% for dic in KEY_REVIEW_LIST %}
                                                {{dic["created"]}} &nbsp;&nbsp;
                                                {{dic["usernm"]}} &nbsp;&nbsp;
                                                {% for i in range( (dic['evelpt'] | int )) %}
                                                    <li><i class="fas fa-star"></i></li>
                                                {% endfor %}
                                                <br><font color="#6666CC"><b>{{dic["tag"]}}</b></font>
                                                <br>{{dic['evelcontent']}} <span></span></li>
                                                <hr>
                                            {% endfor %}
                                        <b><a href="#">[더보기]</a></b>
                                    </div>
                            </div>
                        </div>
```

### 4.5 AJAX

```js
<script>

$(document).ready(function(){

    $(document).on("click",".btn-wishlist",function() {
        alert($(this).attr("value"))
        $.ajax({
			  url: "/booking_add" ,
			  data: {"contentsid":$(this).attr("value")} ,
			  contentType : "application/x-www-form-urlencoded; charset=UTF-8",
			  dataType    : "text" ,
              success: function( resString ) {           //콜백함수처리 방식 : JSON 객체로 들어온다
                    console.log(resString)               //200
              },
              error: function( result ) {
                    alert("error");                     //403, 500
              },
        });
    });


 });
</script>
```

