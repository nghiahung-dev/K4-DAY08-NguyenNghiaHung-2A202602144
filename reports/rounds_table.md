# Bảng so sánh các vòng

Tập kiểm thử: 20 ảnh, 403 box tham chiếu (bỏ qua 14 box cao dưới 16 px). Ngưỡng IoU 0.5; P, R, F1 tính tại conf 0.25.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolo11x cold start (COCO car+bus+truck) | 0 | 0 | 0.904 | — | 1.000 | 0.645 | 0.784 | 0.045 | 0.733 | 0.976 |
| 1 | yolo11x fine-tune vong 1..1 | 12 | 240 | 0.890 | -0.014 | 1.000 | 0.628 | 0.771 | 0.045 | 0.716 | 0.927 |
