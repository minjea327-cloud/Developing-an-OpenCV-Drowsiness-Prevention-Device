## 개요
## 실험장비
1. 라즈베리파이
2. 브레드보드
3. 능동부저
4. 점퍼 케이블 2개
   
## 실험내용

OpenCV의 원리를 이해하고, 능동부저를 이용해 OpenCV 졸음방지 디바이스를 구현하는 것을 목적으로 한다. Python 프로그램으로 눈이 1개 이하로 떠 있을 때는 부저가 울리고 2개 이상 떠 있을 때의 부저가 꺼지도록 설정하여 눈이 감기면 부저가 경고음을 울리는 능력을 습득한다. 더불어 본 실험을 기반으로 향후 실제 차량 내 환경에 적용할 수 있는 스마트 졸음운전 방지 시스템으로 확장 가능한 구조를 이해하고자 한다.
## 코드설명
```
import cv2
from gpiozero import Buzzer
import time

buzzerPin = Buzzer(16)

def main():
    camera = cv2.VideoCapture(-1)
    camera.set(3,640)
    camera.set(4,480)
    
    face_xml = cv2.data.haarcascades + 'haarcascade_frontalface_default.xml'
    eye_xml = cv2.data.haarcascades + 'haarcascade_eye.xml'
    face_cascade = cv2.CascadeClassifier(face_xml)
    eye_cascade = cv2.CascadeClassifier(eye_xml)
    
    while( camera.isOpened() ):
        _, image = camera.read()
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

        faces = face_cascade.detectMultiScale(gray,scaleFactor=1.1,minNeighbors=5,minSize=(100,100),flags=cv2.CASCADE_SCALE_IMAGE)
        print("faces detected Number: " + str(len(faces)))

        if len(faces):
            for (x,y,w,h) in faces:
                cv2.rectangle(image,(x,y),(x+w,y+h),(255,0,0),2)
                
                face_gray = gray[y:y+h, x:x+w]
                face_color = image[y:y+h, x:x+w]
                
                eyes = eye_cascade.detectMultiScale(face_gray,scaleFactor=1.1,minNeighbors=5)
                
                if len(eyes) <= 1:
                    buzzerPin.on()
                else:
                    buzzerPin.off()
                
                for (ex,ey,ew,eh) in eyes:
                    cv2.rectangle(face_color, (ex, ey), (ex+ew, ey+eh), (0,255,0), 2)
        
        cv2.imshow('result', image)
        
        if cv2.waitKey(1) == ord('q'):
            break
    
    cv2.destroyAllWindows()
    buzzerPin.off()

if __name__ == '__main__':
    main()
```
1. 코드 핵심 설명
   
- cv2.VideoCapture(-1) / set(): 카메라를 켜고 화면 크기를 가로 640, 세로 480 해상도로 설정합니다.

- CascadeClassifier: OpenCV에서 제공하는 뼈대 알고리즘(Haar Cascade)을 로드합니다. 각각 얼굴 감지용과 눈 감지용 템플릿입니다.

- cv2.cvtColor(..., cv2.COLOR_BGR2GRAY): 컴퓨터가 빠르게 연산할 수 있도록 카메라 영상을 흑백(그레이스케일)으로 변환합니다.

- detectMultiScale(faces): 흑백 화면에서 얼굴을 찾아 사각형 좌표(x, y, w, h)를 반환합니다. 얼굴이 발견되면 파란색 사각형((255,0,0))을 그립니다.

- face_gray = gray[...]: 찾은 얼굴 영역 내부에서만 눈을 다시 탐색하여 연산 속도를 높입니다. 눈이 발견되면 초록색 사각형((0,255,0))을 그립니다.

- if len(eyes) <= 1: (핵심 로직):

   - 감지된 눈의 개수가 1개 이하(눈을 감았거나, 졸아서 고개가 꺾인 상황 등)라면 부저를 울립니다(buzzerPin.on()).

   - 두 눈이 멀쩡히 잘 보이면 부저를 끕니다(buzzerPin.off()).

- cv2.waitKey(1) == ord('q'): 키보드 'q' 키를 누르면 프로그램이 안전하게 종료됩니다.

2. 출력 결과
   
프로그램을 실행하면 크게 모니터 화면 출력과 터미널 텍스트 출력, 그리고 하드웨어 반응 3가지 결과가 나타납니다.

① 모니터 화면 (윈도우 창)

result라는 이름의 실시간 카메라 창이 뜹니다.

화면에 사람이 나타나면 얼굴 주변에는 파란색 네모, 두 눈 주변에는 초록색 네모가 그려져 따라다닙니다.

② 터미널 창 (텍스트 콘솔)

실시간으로 감지된 얼굴의 개수가 계속해서 프린트됩니다.

```
faces detected Number: 0   <-- 화면에 사람이 없을 때
faces detected Number: 0
faces detected Number: 1   <-- 사람이 등장했을 때
faces detected Number: 1
```

③ 하드웨어 작동 (부저)

평상시 (두 눈을 뜨고 있을 때): 부저가 울리지 않고 조용합니다.

이상 상황 (눈을 감거나, 손으로 한쪽 눈을 가릴 때): 눈 인식 개수가 1개 이하가 되면서 라즈베리 파이 16번 핀에 연결된 부저에서 "삐~" 하고 경고음이 울립니다.

## Opencv 원리

OpenCV(Open Source Computer Vision Library)는 컴퓨터가 인간처럼 이미지나 영상을 보고 이해할 수 있도록 돕는 오픈소스 라이브러리이다. OpenCV는 먼저 이미지를 0부터 255 사이의 숫자로 이루어진 격자 행렬로 변환하여 인식한다. 그 후 이미지 위로 작은 필터를 굴리며 픽셀 값을 계산하는 합성곱(Convolution) 연산을 통해, 이웃한 픽셀 간의 밝기 차이를 파악하고 사물의 테두리(Edge)를 선명하게 추출해낸다. 여기에 특정 밝기를 기준으로 이미지를 흑백으로 나누는 이진화 작업을 거치면 배경과 원하는 사물이 완벽히 분리된다. 마지막으로 크기가 변하거나 회전해도 달라지지 않는 고유한 모서리(특징점)를 찾아 벡터 데이터로 비교함으로써 대상이 무엇인지 최종적으로 인식하게 된다.

----------------
유튜브 데모영상: https://youtu.be/_FDrPahikYM 
