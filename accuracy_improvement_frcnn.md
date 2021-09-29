# 精度优化思路分析

本小节侧重展示在模型迭代过程中优化精度的思路，在本案例中，有些优化策略获得了精度收益，而有些没有。在其他场景中，可根据实际情况尝试这些优化策略。

## (1) 基线模型选择

相较于单阶段检测模型，二阶段检测模型的精度更高但是速度更慢。考虑到是部署到GPU端，本案例选择二阶段检测模型FasterRCNN作为基线模型，其骨干网络选择ResNet50_vd。训练完成后，模型在验证集上图片级别的召回率为96.7%，图片级别的误检率为8.1%。

【名词解释】

* 图片级别的召回率：只要在有目标的图片上检测出目标（不论框的个数），该图片被认为召回。批量有目标图片中被召回图片所占的比例，即为图片级别的召回率。
* 图片级别的误检率：只要在无目标的图片上检测出目标（不论框的个数），该图片被认为误检。批量无目标图片中被误检图片所占的比例，即为图片级别的误检率。

## (2) 基线模型效果分析与优化

| 模型                | Recall（图片级别的召回率） | Error Rate（图片级别的误检率） |
| ------------------- | -------------------------- | ------------------------------ |
| FasterRCNN+ResNet50 | 96.7%                      | 8.1%                           |

由于烟雾和火灾的特殊性，无法精准的计算mAP，因此本案例采用图片级别的召回率和图片级别的误检率作为最终指标。

从分析表格中可以看出，对于没有烟雾和火灾的负样本，存在较严重的误检情况。因此，将验证集上的预测结果进行了可视化，发现由于自然图像中的火焰和烟雾形状非常不规则，导致网络很难提取到非常准确的特征，很多预测框位置并不准确，同时容易将负样本或背景中与火灾烟雾类似的目标误检测成火灾或烟雾。为了解决该问题，选择在骨干网络ResNet50_vd中使用可变形卷积（DCN）。重新训练后，图片级别的误检率降为7.6%，但漏检有了一定增加，图片级别的召回率变为94.5%。

## (3) 数据增强选择

| 训练预处理                                                   | 验证预处理                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| RandomResizeByShort(short_sizes=[640, 672, 704, 736, 768, 800], max_size=1333, interp='CUBIC') | RandomResizeByShort(short_sizes=800,max_size=1333, interp='CUBIC') |
| RandomHorizontalFlip()                                       | Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]) |
| Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]) |                                                              |

在加入了[RandomResizeByShort](https://github.com/PaddlePaddle/PaddleX/blob/dc14077a3b0e8ab9e512a78938d8f01b969f0942/paddlex/cv/transforms/operators.py#L394)、[RandomHorizontalFlip](https://paddlex.readthedocs.io/zh_CN/develop/apis/transforms/det_transforms.html#randomhorizontalflip)这两种数据增强方法后，对模型的优化是有一定的积极作用了，在取消这些预处理后，模型性能会有一定的下降。

**PS**：建议在训练初期都加上这些预处理方法，到后期模型超参数以及相关结构确定最优之后，再进行数据方面的再优化: 比如数据清洗，数据预处理方法筛选等。

## (4) 背景图片加入

本案例将数据集中提供的背景图片按9:1切分成了1116张、135张两部分，并使用(3)中训练好的模型在135张背景图片上进行测试，发现图片级误检率高达21.5%。为了降低模型的误检率，使用[paddlex.datasets.VOCDetection.add_negative_samples](https://paddlex.readthedocs.io/zh_CN/develop/apis/datasets.html#add-negative-samples)接口将1116张背景图片加入到原本的训练集中，重新训练后图片级误检率降低至4%。为了不让训练被背景图片主导，本案例通过将`train_list.txt`中的文件路径多写了一遍，从而增加有目标图片的占比。

| 模型                          | 图片级召回率 | 图片级误检率 |
| ----------------------------- | ------------ | ------------ |
| FasterRCNN+ResNet50           |              |              |
| FasterRCNN+ResNet50+ 背景图片 |              |              |