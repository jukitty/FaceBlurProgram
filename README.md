# FaceBlurProgram
> 사용자로부터 영상을 입력받아 얼굴을 검출해내고, 사용자가 선택한 얼굴 외에 모자이크 처리하는 프로그램이다.  
> yolov5-face와 pretrained된 모델(yolov5s-face)을 사용하여 얼굴을 검출한다.  
> strong_sort를 사용하여 검출된 얼굴을 트래킹하여 하나의 객체로 인식한다.  
> pyside6를 이용하여 GUI를 구성하고, 모자이크가 처리가 완료된 영상을 출력한다.  
  
## How to use
yolov5-face 구동을 위한 pytorch를 설치한다.  
환경에 맞게 설치하되, 본 프로젝트의 개발 환경에 맞는 설치는 다음과 같다.
```
pip install torch==1.11.0+cu113 torchvision==0.12.0+cu113 torchaudio==0.11.0 --extra-index-url https://download.pytorch.org/whl/cu113
```  

다음으로 코드와 yolov5-face, strong_sort와 관련된 의존성 패키지들을 설치한다.
```
git clone https://github.com/YUYUJIN/FaceBlurProgram.git
cd FaceBlurProgram
pip install -r requirements.txt
```  

GUI와 영상 처리와 관련된 패키지들을 추가로 설치한다.
```
pip install pyside6 moviepy
```  
moviepy는 버전문제가 발생하여 포함된 코드로 패키지를 교체한다.

이후 메인코드를 실행한다.
```
python main.py
```

## Yolov5-face
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/yolov5-face.png></img>   
github: https://github.com/deepcam-cn/yolov5-face  

<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/faceData.png></img>  
link: https://www.kaggle.com/datasets/andrewmvd/face-mask-detection  

위 데이터로 마스크 유무, 여러 성별, 인종에 대한 Yolov5-face 성능 검증을 실시하였다. yolov5s 모델 외에도 yolov5m 등의 모델도 확인하였으나 성능차이는 미비하였고, 모두 mAP 50-95가 0.9 이상이었다. 또한 유튜브 영상에 대한 성능 검증에서도 육안으로 확인하였을 때 높은 정확도를 보이는 것을 확인하였다.  

<details>
<summary>Trouble shooting</summary>

<details>
<summary>구조 문제</summary>

yolov5-face 모델로 얼굴 검출 후 결과를 strong_sort 모델에 입력으로 사용하기 위해서는 strong_sort에 포함된 yolov5모델을 교체해야하였다.  
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/face_t.png></img>  
위 이미지와 같이 구조에서의 차이가 존재하였고, strong_sort는 기존의 yolov5를 대상으로 만들어진 코드이므로 서로 코드 내 의존성이 맞지 않았다. 따라서 기존 strong_sort에서 대상이 된 yolov5 특정 버전의 코드에서 yolov5-face와 대비되는 부분과 문제가 되는 부분을 찾아 수정하여 이식하였다.
</details>
<details>
<summary>바운딩 박스</summary>

얼굴이 검출된 영역으로 트래킹을 시도하면 트래킹 id가 튀는 현상이 발생하였다. 검출된 얼굴 영역이 너무 작아 특징점을 도출하기 어려워 새로운 객체로 인식한다고 판단하고 아래와 같이 작업하였다.  
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/boundingbox.PNG></img>  
위와 같이 strong_sort의 입력으로 사용될 얼굴 검출 영역을 확장하여 사용하여 트래킹 결과를 향상하였다.
</details>
</details>

## StrongSort
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/strongsort.png></img>  
github: https://github.com/bharath5673/StrongSORT-YOLO  

트래킹 기술로는 DeepSort와 StrongSort 중 조금더 복잡하지만 성능이 높은 StrongSort를 사용하였다.  
StrongSort는 Kalman 필터 알고리즘와 CNN 모델을 통한 특징 추출을 사용해 matching 방식을 사용하는 방식을 같지만, DeepSort보다 보강된 형태로 사용하였다.  
<img src=></img><(sort)이미지>  
<details>
<summary>Trouble shooting</summary>

<details>
<summary>보강 알고리즘</summary>

Yolov5-face에서 검출된 얼굴 영역을 확장해 사용하여도 트래킹을 놓치는 경우가 많았다. 이를 보완하기 위해 추가적인 작업을 진행하였다.  
검출된 영역의 주파수 영역 해석을 이용하였다. StrongSort로 기존에 탐지된 객체로 인식이 된다면 추가 알고리즘을 거치게된다.  
영역에 대해 공간 주파수 히스토그램을 opencv을 이용해 계산한다. 이후 같은 그룹으로 저장된 영상과 유사도 비교를 진행한다. 유사도는 상관관계, 비타차야 거리, 카이제곱, 교차 검증을 진행하고 네 개의 유사도로 점수를 계산하여 유효 점수를 넘지 못하면 그룹에서 탈락하게 된다.  
참고 자료: https://bkshin.tistory.com/entry/OpenCV-12-%EC%9D%B4%EB%AF%B8%EC%A7%80-%EC%9C%A0%EC%82%AC%EB%8F%84-%EB%B9%84%EA%B5%90-%EC%82%AC%EB%9E%8C-%EC%96%BC%EA%B5%B4%EA%B3%BC-%ED%95%B4%EA%B3%A8-%ED%95%A9%EC%84%B1-%EB%AA%A8%EC%85%98-%EA%B0%90%EC%A7%80-CCTV  
</details>
</details>

## PySide6
GUI 구성을 위해 PySide6를 사용하였다. 추가로 요구 GUI와 유사한 형태를 가진 코드를 참고하여 작업하였다.  
github: https://github.com/Wanderson-Magalhaes/Modern_GUI_PyDracula_PySide6_or_PyQt6  

<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/gui1.png></img>  
1. 탭메뉴: 파일 찾기, 편집, 저장 탭으로 구성  
2. 폴더 탐색기 버튼  
3. 파일 구성 확인용: 폴더 탐색 가능  
4. 메인 화면: 파일 선택 가능  
5. 기능 선택  
6. 편집화면 이동  
  
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/gui2.png></img>  
1. 재생 화면: 처리중인 영상이 스트리밍으로 재생  
2. 얼굴 선택 창: 검출된 얼굴이 실시간으로 나타나고 최종적으로 사용자가 모자이크에서 제외할 얼굴 선택 가능  
3. 얼굴 검출, 얼굴 그룹화 관련 설정 가능  
4. 모자이크 강도 설정 가능  
5. 저장 화면으로 이동  

<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/gui3.png></img>  
1. 최종 영상을 확인할 수 있는 비디오 플레이어  
2. 저장 결로 및 저장 영상 이름 설정  
3. 참고값 출력  
4. 저장 버튼  

<details>
<summary>Trouble shooting</summary>

<details>
<summary>음성 누락/영상 데이터 속도 열화</summary>

opencv로 영상을 처리하다보니 기존 영상의 음성이 누락되는 현상 발생, 추가로 최종 저장된 영상 데이터에서 데이터 속도가 열화되는 현상이 확인되었다.  
이를 해결하기 위해 영상 파일을 opencv외에 moviepy로 데이터를 처리하였다. 음성 데이터는 영상 처리와 다른 흐름으로 최종까지 전달하여 합성하고, 열화는 마지막에 프레임으로 구성된 영상 데이터를 동영상으로 만들 때 비트레이트, 초당 프레임수 등을 조작하여 생성하였다. 
</details>
</details>

## Threading
처리된 영상을 스트리밍하고(동작 반복), GUI는 사용자의 입력을 받는 대기상태로 존재하여야 한다. 이 두 동작은 동시에 실행되어야 한다.  
이를 해결하기 위해 GUI의 동작과 얼굴 검출, 모자이크 처리는 각각 구분된 스레드로 동작하게 하였다.  
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/thread.png></img>  
위와 같이 하나의 객체를 만들고 객체 안에 정의된 메서드들을 스레드로 동작하게하였다.  
<details>
<summary>Trouble shooting</summary>

<details>
<summary>스레드 간 동기화</summary>

메인처리는 GUI 동작을 위해 하위 영상 처리 관련 스레드들의 동작 상태를 참조할 필요가 있다. 이를 위해 영상 처리 동작은 분기별로 메인 스레드의 flag를 확인하면서 동작하게 하고 동작이 끝나면 메인 스레드에게 signal를 보내는 형태로 구현하였다.  
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/thread_t.png></img>  
</details>
</details>

## Example
파일 선택 화면은 위와 동일.  

<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/example1.jpg></img>  

얼굴 검출과 모자이크 동작(영상 편집).  

<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/example2.jpg></img>  

최종 확인 및 저장  

## Reference
Yolov5-face: https://github.com/deepcam-cn/yolov5-face  
Face Mask Dectection: https://www.kaggle.com/datasets/andrewmvd/face-mask-detection  
StrongSort: https://github.com/bharath5673/StrongSORT-YOLO  
Modern_GUI_PyDracula_Pyside6_or_PyQt6: https://github.com/Wanderson-Magalhaes/Modern_GUI_PyDracula_PySide6_or_PyQt6  
OpenCV-12-이미지-유사도-비교-사람-얼굴과-해골-합성-모션-감지-CCTV: https://bkshin.tistory.com/entry/OpenCV-12-%EC%9D%B4%EB%AF%B8%EC%A7%80-%EC%9C%A0%EC%82%AC%EB%8F%84-%EB%B9%84%EA%B5%90-%EC%82%AC%EB%9E%8C-%EC%96%BC%EA%B5%B4%EA%B3%BC-%ED%95%B4%EA%B3%A8-%ED%95%A9%EC%84%B1-%EB%AA%A8%EC%85%98-%EA%B0%90%EC%A7%80-CCTV  

## Produced by 푸루주투
<img src=https://github.com/YUYUJIN/FaceBlurProgram/blob/main/pictures/logo.png style="width:100px; height:100px;"></img>  
team. 푸루주투