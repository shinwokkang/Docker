# 1단계: OS 빈 방 준비 (Ubuntu 22.04 LTS 사용)
FROM ubuntu:22.04
# 2단계: 최신 파이썬(python3) 및 pip 설치
# (-y 옵션을 안 주면 터미널에서 [Y/n]을 물어보며 빌드가 멈춰버리므로 필수입니다!)
RUN apt-get update && apt-get install -y python3 python3-pip
# 3단계: Flask 프레임워크 설치
RUN pip3 install flask
# 4단계: 내 컴퓨터의 app.py를 컨테이너 내부의 /opt/ 폴더로 복사
COPY app.py /opt/app.py
# 5단계: 컨테이너가 켜질 때 실행할 웹 서버 구동 명령어
ENTRYPOINT ["python3", "/opt/app.py"]
