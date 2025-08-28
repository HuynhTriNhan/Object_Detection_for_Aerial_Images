# Few-Shot Object Detection for Aerial Images

## Chủ đề

Nghiên cứu này tập trung vào bài toán Few-Shot Object Detection (FSOD) cho hình ảnh chụp từ trên không (UAV, drone) sử dụng kiến trúc YOLOv11n-OBB với oriented bounding boxes.

## Vấn đề nghiên cứu

### Định nghĩa bài toán

- **Object Detection**: Phát hiện đối tượng, vật thể từ trên cao với góc nhìn từ trên xuống
- **Few-shot training**: Số lượng mẫu huấn luyện thấp (5-10 samples per class)
- **Aerial images**: Ảnh chụp từ vệ tinh, từ drone với góc nhìn từ trên cao

### Thách thức chính

1. **Thiếu dữ liệu**: Ít mẫu huấn luyện cho mỗi class
2. **Domain gap**: Khác biệt giữa DOTA-v1 (pretrained) và YOLODIOR-R (target)
3. **Oriented objects**: Các đối tượng có thể xoay theo nhiều hướng khác nhau
4. **Scale variation**: Kích thước đối tượng thay đổi lớn do độ cao chụp

## Kiến trúc đề xuất

### Base Model

- **YOLOv11n-OBB**: Oriented Bounding Box detection từ Ultralytics, mô hình nano rất phù hợp cài đặt cho drone
- **Pretrained trên DOTA-v1**: 15 classes aerial objects --> phù hợp với bài toán aerial object detection
- **Few-shot fine-tuning trên YOLODIOR-R**: 20 classes với k-shot per class từ dữ liệu DIOR.

### Dataset

#### DOTA-v1 Classes (15 classes - pretrained):

```
plane, ship, storage-tank, baseball-diamond, tennis-court,
basketball-court, ground-track-field, harbor, bridge,
large-vehicle, small-vehicle, helicopter, roundabout,
soccer-ball-field, swimming-pool
```

#### YOLODIOR-R Classes (20 classes - target):

```
airplane, airport, baseballfield, basketballcourt, bridge,
chimney, dam, Expressway-Service-area, Expressway-toll-station,
golffield, groundtrackfield, harbor, overpass, ship, stadium,
storagetank, tenniscourt, trainstation, vehicle, windmill
```

#### Class Mapping

Ánh xạ giữa DOTA-v1 và YOLODIOR-R: có khá nhiều class tương đồng và có một số class novel.

## Phương pháp thực hiện

### 1. Few-Shot Sampling Strategy

- **K-shot per class**: Chọn ngẫu nhiên k samples cho mỗi class.
- **Balanced sampling**: Đảm bảo mỗi class có đủ k samples
- **Multiple seeds**: Thử nghiệm với nhiều random seeds khác nhau
- **Split strategy**: Train (few-shot), Val (train set), Test (original test set)

### 2. Data Augmentation Pipeline

#### Geometric Augmentations

- **Horizontal/Vertical Flip**: p=0.5 cho mỗi loại
- **Rotation**: 90, 180, 270 với p=0.7
- **Letterbox Resize**: Resize về 640x640 giữ tỷ lệ

#### Photometric Augmentations

- **Brightness/Contrast**: alpha in [0.85, 1.15], beta in [-25, 25]
- **Gaussian Blur**: kernel size 3 hoặc 5
- **Gaussian Noise**: sigma in [5, 15]

#### Tác dụng của augmentation

- Tăng diversity cho few-shot data
- Tăng tính orientation cho aerial images
- Tăng tính generalization với limited samples
- Augmentation factor: 3x (tạo 3 augmented versions cho mỗi original image)

### 3. Training Configuration

#### Few-Shot Learning Strategy

- **Low Learning Rate**: 0.001-0.002 để tránh catastrophic forgetting
- **Layer Freezing**: Freeze backbone layers tùy theo k-shot
- **Early Stopping**: Monitor validation để tránh overfitting
- **Cosine Learning Rate**: Giảm learning rate theo cosine schedule

#### Hyperparameters theo k-shot

- **5-shot**: lr=0.001, epochs=100, freeze=10 layers, patience=15
- **10-shot**: lr=0.002, epochs=80, freeze=5 layers, patience=10
- **Batch size**: 16
- **Image size**: 640x640
- **Optimizer**: AdamW với weight decay=0.0005

## Kết quả thực nghiệm

### Thiết lập thí nghiệm

- **K-shots**: 5-shot, 10-shot per class
- **Seeds**: 1 random seed (seed=0)
- **Conditions**: Baseline (no augmentation) vs Augmented
- **Metrics**: mAP50, mAP50-95, Precision, Recall

### Kết quả chính

#### Performance Summary

| K-Shot  | Condition | mAP50  | mAP50-95 | Precision | Recall |
| ------- | --------- | ------ | -------- | --------- | ------ |
| 5-shot  | Baseline  | 0.5326 | 0.3934   | 0.6085    | 0.5289 |
| 5-shot  | Augmented | 0.5629 | 0.4219   | 0.6641    | 0.5166 |
| 10-shot | Baseline  | 0.6199 | 0.4520   | 0.7155    | 0.5757 |
| 10-shot | Augmented | 0.6420 | 0.4770   | 0.7253    | 0.5915 |

#### Augmentation Impact

- **5-shot**: +5.68% improvement với augmentation
- **10-shot**: +3.55% improvement với augmentation

### Quan sát và phân tích

1. **Augmentation hiệu quả hơn ở few-shot**: Cải thiện 5.68% cho 5-shot vs 3.55% cho 10-shot
2. **Performance tăng theo k-shot**: 10-shot đạt mAP50 cao hơn 5-shot đáng kể
3. **Precision cao hơn Recall**: Model có xu hướng conservative trong prediction
4. **mAP50-95 thấp hơn mAP50**: Cho thấy localization chưa chính xác ở IoU cao
