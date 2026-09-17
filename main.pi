# -*- coding: utf-8 -*-
"""
어제의 영화 박스오피스를 보여주는 스트림릿(Streamlit) 앱입니다.
- 데이터 출처: KOBIS(영화진흥위원회) 오픈API
- 초보자를 위해 각 단계마다 한국어 주석을 달아두었습니다.
"""

import requests               # 외부 API(KOBIS)에 웹 요청을 보내기 위한 라이브러리
import pandas as pd            # 표 형태(데이터프레임)로 데이터를 다루기 위한 라이브러리
import streamlit as st         # 웹 화면을 그려주는 라이브러리
from datetime import datetime, timedelta  # 날짜/시간 계산용
from zoneinfo import ZoneInfo  # 시간대(타임존)를 다루기 위한 표준 라이브러리 (파이썬 3.9+)


# -----------------------------------------------------------------------
# 1. 기본 설정
# -----------------------------------------------------------------------

# 스트림릿 페이지의 기본 설정(제목, 아이콘, 레이아웃)을 지정합니다.
st.set_page_config(
    page_title="어제의 박스오피스",
    page_icon="🎬",
    layout="wide",
)

# KOBIS 오픈API의 "일별 박스오피스" 조회 주소입니다.
KOBIS_URL = "https://www.kobis.or.kr/kobisopenapi/webservice/rest/boxoffice/searchDailyBoxOfficeList.json"


def get_yesterday_kst_str() -> str:
    """
    한국 시간(KST) 기준으로 '어제' 날짜를 yyyymmdd 형식의 문자열로 돌려줍니다.

    주의:
    - 스트림릿 클라우드 서버의 시계는 한국 시간이 아닐 수 있습니다.
      (보통 UTC를 사용합니다.)
    - 그래서 항상 datetime.now(ZoneInfo("Asia/Seoul"))처럼
      '한국 시간대'를 명시해서 현재 시각을 구해야 합니다.
    - 오늘 날짜에서 하루(1일)를 빼서 '어제'를 계산합니다.
    """
    now_kst = datetime.now(ZoneInfo("Asia/Seoul"))
    yesterday_kst = now_kst - timedelta(days=1)
    return yesterday_kst.strftime("%Y%m%d")


# -----------------------------------------------------------------------
# 2. API 호출 함수 (1시간 동안 결과를 기억(캐시)해서 같은 날짜는 다시 부르지 않음)
# -----------------------------------------------------------------------

@st.cache_data(ttl=3600)  # ttl=3600초 = 1시간 동안 같은 target_dt 결과를 재사용합니다.
def fetch_box_office(target_dt: str, api_key: str):
    """
    KOBIS 오픈API에 요청을 보내서 target_dt(예: '20240101') 날짜의
    일별 박스오피스 목록을 가져옵니다.

    반환값은 항상 다음과 같은 딕셔너리 형태입니다.
    {
        "ok": True/False,          # 성공 여부
        "error_message": str,      # 실패 시 사용자에게 보여줄 한국어 안내 문구
        "movie_list": list,        # 성공 시 영화 정보 리스트 (dailyBoxOfficeList)
    }
    """
    params = {
        "key": api_key,
        "targetDt": target_dt,
    }

    # (1) 네트워크 요청 자체가 실패하는 경우 (인터넷 문제, 서버 다운, 시간 초과 등)
    try:
        response = requests.get(KOBIS_URL, params=params, timeout=10)
    except requests.exceptions.RequestException:
        return {
            "ok": False,
            "error_message": (
                "KOBIS 서버에 접속하지 못했습니다. "
                "인터넷 연결 상태나 KOBIS 서버 상태를 확인해 주세요."
            ),
            "movie_list": [],
        }

    # (2) HTTP 상태 코드가 200이 아닌 경우 (예: 404, 500 등)
    if response.status_code != 200:
        return {
            "ok": False,
            "error_message": (
                f"KOBIS 서버가 오류 응답(상태 코드 {response.status_code})을 보냈습니다. "
                "잠시 후 다시 시도해 주세요."
            ),
            "movie_list": [],
        }

    # (3) 응답 내용이 올바른 JSON이 아닌 경우
    try:
        data = response.json()
    except ValueError:
        return {
            "ok": False,
            "error_message": (
                "KOBIS 서버 응답을 해석할 수 없습니다(JSON 형식 아님). "
                "요청 주소나 파라미터가 올바른지 확인해 주세요."
            ),
            "movie_list": [],
        }

    # (4) 인증키가 틀렸거나 요청이 잘못된 경우, 상태코드는 200이지만
    #     'faultInfo' 상자가 대신 들어옵니다.
    if "faultInfo" in data:
        fault = data["faultInfo"]
        fault_message = fault.get("message", "알 수 없는 오류")
        return {
            "ok": False,
            "error_message": (
                f"KOBIS API가 오류를 반환했습니다: {fault_message}\n"
                "→ secrets에 등록한 KOBIS_KEY(인증키) 값이 정확한지, "
                "그리고 조회 날짜(targetDt) 형식이 8자리 숫자(yyyymmdd)인지 확인해 주세요."
            ),
            "movie_list": [],
        }

    # (5) 정상 응답이어야 할 boxOfficeResult가 아예 없는 경우 (예상 못한 구조)
    if "boxOfficeResult" not in data:
        return {
            "ok": False,
            "error_message": (
                "KOBIS 응답 구조가 예상과 다릅니다. "
                "KOBIS 공식 문서에서 응답 형식이 바뀌지 않았는지 확인해 주세요."
            ),
            "movie_list": [],
        }

    movie_list = data["boxOfficeResult"].get("dailyBoxOfficeList", [])

    # (6) 영화 목록이 비어 있는 경우 (예: 아직 해당 날짜 집계가 안 됐거나, 데이터가 없는 날)
    if not movie_list:
        return {
            "ok": False,
            "error_message": (
                "해당 날짜의 박스오피스 데이터가 비어 있습니다. "
                "조회 날짜가 너무 이르거나(집계 전), KOBIS 서버 점검 중일 수 있습니다."
            ),
            "movie_list": [],
        }

    # 여기까지 왔다면 정상적으로 데이터를 받아온 것입니다.
    return {
        "ok": True,
        "error_message": "",
        "movie_list": movie_list,
    }


# -----------------------------------------------------------------------
# 3. 데이터 가공 함수
# -----------------------------------------------------------------------

def build_dataframe(movie_list: list) -> pd.DataFrame:
    """
    API에서 받은 movie_list(딕셔너리 리스트)를 화면에 보여주기 좋은
    데이터프레임(표)으로 바꿔줍니다.

    KOBIS API는 숫자 값도 전부 '문자열'로 내려주기 때문에,
    정렬과 그래프에 쓰려면 반드시 숫자(int) 타입으로 바꿔줘야 합니다.
    """
    df = pd.DataFrame(movie_list)

    # 문자열로 온 숫자 컬럼들을 정수형으로 변환합니다.
    # errors="coerce"로 혹시 이상한 값이 있으면 NaN 처리 후 0으로 채웁니다.
    numeric_columns = ["rank", "audiCnt", "audiAcc", "scrnCnt", "showCnt"]
    for col in numeric_columns:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors="coerce").fillna(0).astype(int)

    # 순위(rank) 기준으로 오름차순 정렬합니다.
    df = df.sort_values(by="rank").reset_index(drop=True)

    return df


# -----------------------------------------------------------------------
# 4. 화면 그리기
# -----------------------------------------------------------------------

def main():
    st.title("🎬 어제의 박스오피스")

    # (1) secrets 금고에서 API 키를 꺼내옵니다. 코드에는 절대 키를 직접 쓰지 않습니다.
    #     스트림릿 클라우드에서는 앱 설정 > Secrets 메뉴에 아래처럼 등록해두면 됩니다.
    #     KOBIS_KEY = "발급받은_인증키"
    if "KOBIS_KEY" not in st.secrets:
        st.error(
            "secrets에 KOBIS_KEY가 설정되어 있지 않습니다.\n"
            "→ 스트림릿 클라우드의 'Settings > Secrets'에서 "
            "KOBIS_KEY = \"발급받은 인증키\" 형태로 등록해 주세요."
        )
        return

    api_key = st.secrets["KOBIS_KEY"]

    # (2) 한국 시간 기준 '어제' 날짜를 계산합니다.
    target_dt = get_yesterday_kst_str()
    # 화면에 보여줄 때는 사람이 읽기 좋게 yyyy-mm-dd 형태로 바꿔줍니다.
    pretty_date = f"{target_dt[0:4]}-{target_dt[4:6]}-{target_dt[6:8]}"
    st.caption(f"조회 기준일(한국 시간, 어제): {pretty_date}")

    # (3) API 호출 (같은 날짜라면 1시간 동안은 캐시된 결과를 재사용합니다.)
    result = fetch_box_office(target_dt, api_key)

    # (4) 실패했다면, 빈 화면 대신 안내 메시지를 보여주고 함수를 끝냅니다.
    if not result["ok"]:
        st.error(result["error_message"])
        return

    # (5) 성공했다면 데이터프레임으로 가공합니다.
    df = build_dataframe(result["movie_list"])

    # ---------------------------------------------------------------
    # 5-1. 1위 영화를 지표 카드 3장으로 크게 보여주기
    # ---------------------------------------------------------------
    top1 = df.iloc[0]  # rank 기준으로 이미 정렬했으므로 첫 번째 행이 1위입니다.

    st.subheader(f"👑 1위: {top1['movieNm']}  (개봉일: {top1['openDt']})")

    col1, col2, col3 = st.columns(3)
    col1.metric("어제 관객수", f"{top1['audiCnt']:,}명")
    col2.metric("누적 관객수", f"{top1['audiAcc']:,}명")
    col3.metric("스크린수", f"{top1['scrnCnt']:,}개")

    st.divider()

    # ---------------------------------------------------------------
    # 5-2. 관객수 상위 5편 막대그래프
    # ---------------------------------------------------------------
    st.subheader("📊 관객수 상위 5편")

    top5 = df.sort_values(by="audiCnt", ascending=False).head(5)
    # st.bar_chart는 인덱스를 x축, 값을 y축으로 그려줍니다.
    chart_data = top5.set_index("movieNm")[["audiCnt"]]
    st.bar_chart(chart_data)

    st.divider()

    # ---------------------------------------------------------------
    # 5-3. 전체 표 보여주기
    # ---------------------------------------------------------------
    st.subheader("📋 전체 순위표")

    # 화면에 보여줄 컬럼만 골라서, 한글 컬럼명으로 바꿔줍니다.
    table_df = df[["rank", "movieNm", "openDt", "audiCnt", "audiAcc", "scrnCnt"]].rename(
        columns={
            "rank": "순위",
            "movieNm": "영화명",
            "openDt": "개봉일",
            "audiCnt": "관객수",
            "audiAcc": "누적관객",
            "scrnCnt": "스크린수",
        }
    )

    st.dataframe(table_df, use_container_width=True, hide_index=True)


# 스트림릿 앱은 이 파일이 직접 실행될 때 main()을 호출합니다.
if __name__ == "__main__":
    main()
