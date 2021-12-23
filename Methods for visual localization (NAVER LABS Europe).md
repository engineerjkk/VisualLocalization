# Visual localization
- 이미지로부터 현재 위치를 추정하는 기술.
- 로보틱스, 자율주행, 증강현실
- 기존에 사용하던 방법: GNSS
  - GNSS는 실내에서 사용 불가능
  - 오차 범위가 미터단위로 큼
  - 방향 정보가 없음
    -GPS와 Compass 정보를 합성하여 1~2m 정도의 위치 오차와 10도 정도의 방향 오차로 현재 위치를 추정할 수 있으나, 로보틱스/자율주행/증강현실에서 요구하는 정확도에 훨씬 못 미침.
 

# 어려운 이유
- Visual localization을 통해 실내와 실외 환경에서 cm 단위로 위치를 추정할 수 있어야함.
  - 실외: 자율주행
  - 실내: 로보틱스, 증강현실, 자율주행 (GNSS가 닿지 않는 주차장 등)
  - 사전에 모아둔 reference 이미지들과 내가 현재 보고 있는 query 이미지들간의 correspondence를 이용하여 현재 위치를 추정 가능함.
    -이러한 ‘visual map’ 정보는 3D reconstruction을 통해 생성 가능함.
- 기술적 어려움:
  - 조명 변화.
    -Visual map이 낮에 찍은 사진들로 이뤄졌다면, 밤에 찍은 query 이미지로 위치를 추정할 수 있을까?
  - 동적 객체
  - Visual map을 만들 때 움직이는 객체가 포함되어있었다면?
  - 날씨/계절 변화
  - 가려짐 / 부분적 가려짐
  - 시점의 차이
- 다양한 파이프라인이 제안 됨.
- 다양한 데이터셋이 제안됨.
    - 데이터셋마다 포맷이 많이 다름.
    - NAVER LABS에서 만든 Kapture를 통해 포맷을 하나로 통합함.
 

# 기술 오버뷰
1. Structure-based
  - 오래된 방법
  - 3D reconstruction으로 point cloud를 생성하고, local feture matching을 통해 query image와 3d map같의 correspondence를 찾는 작업.
2. Structure-based + Image retrieval
  - Image retrieval 등을 사용해서 search space를 줄일 수 있음.
3. Pose interpolation
  - 3D reconstruction을 수행하지 않고, reference image로부터 pose interpolation을 함.
4. Relative pose estimation
  - 3D reconstruction을 수행하지 않고, reference image로부터 query image에 대한 relative pose estimation을 함.
5. Scene poitn regression
  - 딥러닝을 사용하여 2D pixel location과 3D point의 correspondence를 구함.
  - Structured-based 방식과 비슷하지만, 딥러닝을 사용함.
  - 학습 과정에서 3D reconstruction을 사용하기도 함.
6. Absolute pose regression
  - End-to-end로 이미지->pose regression을 수행함.
- 각각의 방법들은 generalization과 정확도가 많이 다름.

![image](https://user-images.githubusercontent.com/76835313/147192476-483d79a9-e16b-4fb2-ae38-0ce1e4c205a3.png)
![image](https://user-images.githubusercontent.com/76835313/147192480-e0ea8a6d-3e17-42e9-815b-5df15c36c6ee.png)


# Structure-based methods
- 3D reconstruction을 사용하면 정확도가 높아진다.
    -하지만 종종 3D reconstruction을 할 수 없거나, 3D map을 유지보수하기 어려운 상황이 있다.
- 3D reconstruction은 Structure-from-Motion(SfM)으로 수행한다.
  - SfM은 여러개의 이미지로부터 local feature correspondence를 기반으로 reconstruction을 수행한다.
  - Local feature correspondence는 feature descriptor들간의 거리를 기반으로 만들어진다. 
    - 하지만 이 feature descriptor들은 사실 visual localization을 위해 만들어진 것이 아니기 때문에 여러 단점을 가지고 있다.
    - 낮/밤, 계절, 날씨를 구분하지 못한다는 것들이다.
    - 최근에는 딥러닝 기반으로 학습한 local feature를 사용하기도 한다.
  - 3D reconstruction은 local feature들간의 2D-2D correspondence를 통해 생성된다.
- Query 이미지의 위치를 계산하기 위해서는, query 이미지와 3D map간의 2D-3D correspondence를 descriptor matching으로 찾고, perspective-n-point (PnP) solver + RANSAC을 사용해서 위치 정보를 구한다.
    - Large-scale map의 경우 3D map이 엄청나게 커지게 되며 query 이미지와 3D map에 대한 correspondence를 탐색할 때 엄청난 계산량을 필요로 한다.
      - Aachen-Day-Night 데이터셋의 경우, 3D point는 약 70만개 ~ 250만개 정도 된다고 한다 (셋팅에 따라…).
    - Search space를 줄이기 위해서는 image retrieval이 사용된다.
      -비슷한 이미지를 가진 곳에서만 correspondence를 찾아보는 전략이다.
  - Global map에서 correspondence를 찾는 방법이 아닌, retrieved image들로 local map을 만들어서 그 안에서 correspondence를 구할 수도 있다.
    - 이 경우 우리는 global map을 유지보수하지 않아도 된다.
    - 하지만 retrieved image들로부터 local map을 항상 만들 수 있다는 보장이 없다.
 

# Image retrieval-based methods
- Image retrieval-based 방식을 쓰려면 우선 데이터셋에 해당 이미지에 대한 위치정보 label이 필요하다.
    - GPS 정보나 6DOF 위치 정보로 보통 되어있다.
-  Image retrieval은 DB로부터 가장 비슷한 이미지를 찾는다.
    - 종종 re-ranking step이 함께 사용되기도 한다.
    - ‘가장 비슷한 이미지’에 대한 criteria는 보통 2가지가 있다.
    1. Landmark retrieval - Query와 retrieved 이미지가 동일한 landmark를 많이 가지고 있을 때
    2. Geolocalization / Place recognition - Query와 retrieved 이미지가 비슷한 위치에서 찍었을 때
  - 예전에는 image retrieval을 위해 bag-of-visual-words를 사용했다.
  - 하지만 최근에는 딥러닝으로 좀 더 high-level semantics를 포함하는 정보를 사용한다.
    - Ranking loss를 사용해서 retrieval 작업에 특화된 네트워크를 만들기도 한다.
  - Image retrieval을 사용해서 얻는 이점은 2가지이다.
    1. Structure-based 방식을 사용할 때 search space 줄이기
    2. (Structure가 없을 때) Direct localization으로 pose를 구하기
      - Nearest neighbour image나 k개의 이미지들의 interpolation으로 구할 수 있음.
      - Camera intrinsic이 있다면 relative pose를 구할 수도 있다.
Retrieved image의 absolute pose도 안다면, query image의 absolute pose도 계산할 수 있다.
 

Pose regression-based method
CNN을 이용해서 RGB->pose를 계산하는 end-to-end 방식
Low-level feature에 pose estimation을 위한 정보가 들어있다는 가정을 가짐.
Transfer learning을 쓸 수 있다고 함?
PoseNet의 경우, VGGNet/ResNet의 CNN 레이어와 FC 레이어를 사용해서 6D pose를 regress함. (기존의 VGGNet/ResNet은 FC 레이어 대신 softmax 레이어를 가지고 있음)
Pose regression 방식의 장점:
Feature engineering이 필요없다.
딥러닝이 알아서 robust feature를 학습함 (weather, viewpoint, lighting etc)
Structure-based 방식보다 메모리를 훨씬 적게 사용한다.
모델은 해봤자 50mb. SfM 포인트 클라우드는 Gb 단위.
Transfer learning을 사용해서 중간 크기의 데이터셋에서 잘 쓸 수 있다.
단점:
좀 부정확한 편.
PoseNet의 정확도를 높이기 위해 다음과 같은 방식이 시도되었다.
새로운 loss function 사용
LSTM 레이어 추가
추가 센서 사용
Absolute + relative pose를 동시에 학습 - VLocNet
거기에 semantic segmentation도 같이 수행 - VLocNet++
이러한 방식은 결국 scene에 대한 정보를 compression하는 것에 지나지 않는다.
즉, generalize하기 어렵다.
최근에는 hybrid pose learning 방식이 적용되고 있다.
Learning은 좋은 2D-3D correspondence를 학습하는데에 집중하고, 기존의 structure-based + image retrieval 방식을 사용하던 파이프라인에서 집중하던 geometrical constraint를 그대로 사용하는 방식이 있다.
하지만 이러한 방식은 pose accuracy를 높여주나, 새로운 scene에 generalize되지 않는다.
최근의 SANet의 경우 camera localization을 위한 scene agnostic neural architecture를 제안했다고 한다. (?)
Model parameter와 scene이 독립적이라고 한다 (?)
 

정리
Geometric 방식이 아직 end-to-end 방식보다 좋다.
하지만 최고의 성능을 내기 위해서는 geometric 방식에서 몇가지 모듈을 딥러닝 솔루션으로 바꾸는게 좋다.
예를 들어, local / global feature extractor는 바꾸는게 좋다.
Handcrafted 방식들에 비해 좀 더 다양한 variation에 강인하기 때문이다.
Pure retrieval 기법은 rough location을 얻는데에 굉장히 좋은 솔루션이 될 수 있다.
빠르고 계산량이 적기 때문이다.
Regression pose generalization 분야는 아직 연구가 더 필요하다.
Semantic 정보를 사용하는 시도가 아직 크게 없다.
이러한 시도는 local feature match의 성능을 높이거나
특정 object를 localization에 사용할 수 있다.
반대로, 특정 object를 인식해서 localization에서 제외할 수 있다 (움직이는 물체들).
 

볼만한 논문들
Relja Arandjelovic, Petr Gronat, Akihiko Torii, Tomas Pajdla, and Josef Sivic. NetVLAD: CNN Architecture for Weakly Supervised Place Recognition. In CVPR, 2016.

Eric Brachmann, Alexander Krull, Sebastian Nowozin, Jamie Shotton, Frank Michel, Stefan Gumhold, and Carsten Rother. DSAC – differentiable RANSAC for camera localization. In CVPR, 2017.

Eric Brachmann and Carsten Rother. Learning less is more – 6d camera localization via 3d surface regression. In CVPR, 2018.

Eric Brachmann and Carsten Rother. Expert sample consensus applied to camera re-localization. In ICCV, 2019.

Eric Brachmann and Carsten Rother. Visual camera re-localization from rgb and rgb-d images using dsac, arXiv, 2020.

(Brachmann 최고…)

Gabriela Csurka, Christopher R. Dance, and Martin Humenberger. From Handcrafted to Deep Local Invariant Features. arXiv, 1807.10254, 2018.

Daniel DeTone, Tomasz Malisiewicz, and Andrew Rabinovich. SuperPoint: Self-Supervised Interest Point Detection and Description. In CVPR Workshops, 2018.

Mihai Dusmanu, Ignacio Rocco, Tomas Pajdla, Marc Pollefeys, Josef Sivic, Akihiko Torii, and Torsten Sattler. D2-Net: a Trainable CNN for Joint Description and Detection of Local Features. In CVPR, 2019.

Martin Humenberger, Yohann Cabon, Nicolas Guerin, Julien Morat, Jerome Revaud, Philippe Rerole, Noe Pion, Cesar de Souza, Vincent Leroy, and Gabriela Csurka. Robust Image Retrieval-based Visual Localization using Kapture. arXiv:2007.13867, 2020.

Revaud Jerome, Philippe Weinzaepfel, Cesar De Souza, and Martin Humenberger.R2D2: Reliable and Repeatable Detectors and Descriptors. In NeurIPS, 2019.

Alex Kendall, Matthew Grimes, and Roberto Cipolla. PoseNet: A Convolutional Network for Real-Time 6-DOF Camera Relocalization. In ICCV, 2015.

Alex Kendall and Roberto Cipolla. Geometric Loss Functions for Camera Pose Regression with Deep Learning. In CVPR, 2017.

Xiaotian Li, Shuzhe Wang, Yi Zhao, Jakob Verbeek, and Juho Kannala. Hierarchical scene coordinate classification and regression for visual localization. In CVPR, 2020.

Liu Liu, Hongdong Li, and Yuchao Dai. Efficient Global 2D-3D Matching for CameraLocalization in a Large-Scale 3D Map. In ICCV, 2017.

Noé Pion, Martin Humenberger, Gabriela Csurka, Yohann Cabon, and Torsten Sattler. Benchmarking Image Retrieval for Visual Localization. In 3DV, 2020.

Paul-Edouard Sarlin, Cesar Cadena, Roland Siegwart, and Marcin Dymczyk. From Coarse to Fine: Robust Hierarchical Localization at Large Scale. In CVPR, 2019.

Paul-Edouard Sarlin, Daniel DeTone, Tomasz Malisiewicz, and Andrew Rabinovich. SuperGlue: Learning Feature Matching with Graph Neural Networks. In CVPR, 2020.

Torsten Sattler, Will Maddern, Carl Toft, Akihiko Torii, Lars Hammarstrand, Erik Stenborg, Daniel Safari, Masatoshi Okutomi, Marc Pollefeys, Josef Sivic, Fredrik Kahl, and Tomas Pajdla. Benchmarking 6DoF Outdoor Visual Localization in Changing Conditions. In CVPR, 2018.

Torsten Sattler, Qunjie Zhou, Marc Pollefeys, and Laura Leal-Taixe. Understanding the Limitations of CNN-based Absolute Camera Pose Regression. In CVPR, 2019.

Johannes L. Schonberger and Jan-Michael Frahm. Structure-from-motion Revisited. In CVPR, 2016.

Johannes Lutz Schonberger, Hans Hardmeier, Torsten Sattler, and Marc Pollefeys. Comparative Evaluation of Hand-Crafted and Learned Local Features. In CVPR, 2017.

Matthias Schorghuber, Daniel Steininger, Yohann Cabon, Martin Humenberger, and Margrit Gelautz. Slamantic-leveraging semantics to improve vslam in dynamic environments. In ICCV Workshops, 2019.

H. Taira, M. Okutomi, T. Sattler, M. Cimpoi, M. Pollefeys, J. Sivic, T. Pajdla, and A. Torii. InLoc: Indoor Visual Localization with Dense Matching and View Synthesis. PAMI, 2019.

Philippe Weinzaepfel, Gabriela Csurka, Yohann Cabon, and Martin Humenberger. Visual localization by learning objects-of-interest dense match regression. In CVPR, June 2019.

Luwei Yang, Ziqian Bai, Chengzhou Tang, Honghua Li, Yasutaka Furukawa, and Ping Tan. Sanet: Scene agnostic network for camera localization. In ECCV, 2019.

Qunjie Zhou, Torsten Sattler, Marc Pollefeys, and Laura Leal-Taixe. To Learn or not to Learn: Visual Localization from Essential Matrices. ICRA, 2020.
