# 智能工人\_bdc2320338 队伍算法说明

## 文件目录说明

文件目录如下：

```PowerShell
│
│
└─智能工人_bdc2320338
    │  README.md
    │
    ├─1shot
    │  │  config.yaml
    │  │  config_pretrain.yaml
    │  │
    │  ├─checkpoints
    │  │      model_best.pth
    │  │
    │  └─log_files
    │          ADM-resnet12-train-Dec-01-2023-23-15-46.log
    │          ADM-resnet12-train-Dec-02-2023-11-46-26.log
    │
    └─5shot
        │  config.yaml
        │  config_pretrain.yaml
        │
        ├─checkpoints
        │      model_best.pth
        │
        └─log_files
                DeepBDC-resnet12Bdc-train-Nov-30-2023-08-10-44.log
                DeepBDC_Pretrain-resnet12Bdc-train-Nov-29-2023-19-09-45.log


```

最外层为 1shot、5shot 文件夹和 readme.md

## 关于数据集的说明

在训练一段时间后发现 LibFewShot 库的读取数据集的方式存在一定问题，即在验证和测试的时候用的都是 train.csv 的文件，所以对于其中的 core/data/dataset.py 的部分进行了简单的修改，把原来 99 行中的`_generate_data_list `函数中的

```Python
meta_csv = os.path.join(self.data_root, "{}.csv".format(self.mode))
meta_csv = "./WebCaricature/train.csv"
```

改为了如下代码：

```Python
meta_csv = os.path.join(self.data_root, "{}.csv".format(self.mode))
if self.mode == 'train':
  meta_csv = "./WebCaricature/train.csv"
elif self.mode == 'val':
  meta_csv = "./WebCaricature/val.csv"
elif self.mode == 'test':
  meta_csv = "./WebCaricature/test.csv"
```

下面的所有测试集都是在 WebCaricature 的 test.csv 上进行

## 关于 5way-1shot 算法的说明

主要算法：使用 adm 作为分类器+resnet12 作为 backbone，方式是先进行预训练然后进行微调训练。

使用 LibFewShot 库中的 config/adm.yaml 文件进行预训练和微调训练。

### 预训练阶段

**特别注意：**

- 预训练使用的是修改后的 dataset.py，请复现时候千万注意

- 5-1 预训练阶段没有使用数据增强

- **预训练阶段的 log 文件为 1shot/log_files/ADM-resnet12-train-Dec-01-2023-23-15-46.log**

**预训练的主要参数为（预训练配置为 config_pretrain.yaml，在 1shot 文件夹目录下）：**

```YAML
train_episode: 1000
test_episode: 1000
epoch: 50
initial_lr: 0.01
```

**预训练阶段没有使用数据增强,同时采用了 MultiStepLR 进行动态的学习率调整，初始学习率为 0.01**。

预训练在测试集上达到的效果为：

Test Accuracy: 83.965

Aver Accuracy: 84.498

### 正式训练阶段

**特别注意：**

- 使用的是修改过后的 dataset.py 文件，请复现时候注意。

- 正式训练是之前预训练得到的权重文件来作为预训练模型，然后进行较小学习率的微调。

- **正式训练日志文件为：1shot/log_files/ADM-resnet12-train-Dec-02-2023-11-46-26.log**

- **需要在 adm.yaml(或者是 config.yaml)中加入以下内容：**

```YAML
pretrain_path: ./results/ADM-WebCaricature-resnet12-5-1-Dec-01-2023-23-15-46/checkpoints/emb_func_best.pth
```

请注意，路径是预训练中生成的结果下 checkpoints 中的 emb_func_best.pth

**如果在进行复现的时候运行 config.yaml（在 1shot 目录下），同样需要修改预训练权重的路径。**

**正式训练的主要参数为（正式训练配置文件为 config.yaml，在 1shot 文件夹目录下）：**

```YAML
train_episode: 1000
test_episode: 1000
epoch: 10
initial_lr: 5e-5
augment_times: 2
```

**需要注意的是预训练阶段采用了数据增强，也就是调整了 augment_times 参数，同时采用了 MultiStepLR 进行动态的学习率调整，初始学习率为 5e-5**。

预训练在测试集上达到的效果为：

Test Accuracy: 84.339

&#x20;Aver Accuracy: 84.878

可能在不同测试集上有一定差异。

## 关于 5way-5shot 算法的说明

算法要点：

- 使用 deepbdc 分类器+resnet12BDC 作为 backbone，方式是先预训练然后进行微调训练。

- 使用 LibFewShot 库中的 config/deepbdc_pretrain.yaml 文件进行预训练，使用 config/deepbdc.yaml 文件进行正式训练。（也可采用 5shot 目录下的 pretrain_config.yaml 和 config.yaml 文件）

- **注意在使用 deepbdc 算法时候要改 num_class 为 200**

### 预训练阶段

**特别注意：**

- 使用的是修改过后的 dataset.py 文件，请复现时候注意

- **5way-5shot 采用 deepbdc 算法时候需要在 core/model/finetuning/deepbdc_pretrain.py 中修改一个参数 penalty_C 为 2.0，它在该 py 文件中的 class DeepBDC_Pretrain 中（一定注意）**

- 预训练日志文件为 5shot/log_files/DeepBDC_Pretrain-resnet12Bdc-train-Nov-29-2023-19-09-45.log

**预训练的主要参数为（预训练配置为 config_pretrain.yaml，在 5shot 文件夹目录下）：**

```YAML
train_episode: 1000
test_episode: 1000
epoch: 60
initial_lr: 0.05
augment_times: 2
augment_times_query: 2
test_query: 10
```

**预训练阶段采用了数据增强，同时调整了 augment_times 和 augment_times_query 参数为 2，同时使用了 MultiStepLR 进行动态的学习率调整，初始学习率为 0.05。**

预训练在测试集上达到的效果为：

Test Accuracy: 95.630 Aver Accuracy: 95.526

可能在不同测试集上有一定差异。

### 正式训练

**特别注意：**

- 使用的是修改过后的 dataset.py 文件，请复现时候注意

- 5way-5shot 采用 deepbdc 算法时候需要在 core/model/finetuning/deepbdc_pretrain.py 中修改一个参数 penalty_C 为 2.0，它在该 py 文件中的 class DeepBDC_Pretrain 中

- 正式训练日志文件为 5shot/log_files/DeepBDC-resnet12Bdc-train-Nov-30-2023-08-10-44.log

- **正式训练时候需要修改 config.yaml 里面的预训练的权重的路径**

**正式训练的主要参数为（配置文件为 config.yaml，在 5shot 文件夹目录下）：**

```YAML
train_episode: 1000
test_episode: 1000
epoch: 60
initial_lr: 0.01
augment_times: 2
augment_times_query: 2
test_query: 10
```

**正式训练阶段采用了数据增强，同时调整了 augment_times 和 augment_times_query 参数为 2，同时使用了 MultiStepLR 进行动态的学习率调整，初始学习率为 0.01。**

正式训练在测试集上达到的效果为：

Test Accuracy: 98.698

Aver Accuracy: 98.777

可能在不同测试集上有一定差异。
