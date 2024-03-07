# 智能工人\_bdc2320338 队伍算法说明

## 文件目录说明

文件目录如下：

```PowerShell
│
│
└─bdc2320338_智能工人_国赛复核材料
    │  README.md
    │
    ├─1shot
    │  │  config.yaml
    │  │  config_pretrain.yaml
    │  │
    │  ├─checkpoints
    │  │      model_best.pth
    │  │      emb_func_best.pth
    │  │
    │  └─log_files
    │          ADM-resnet12-train-Dec-02-2023-13-55-09.log
    │          ADM-resnet12-train-Dec-04-2023-23-48-27.log
    │
    └─5shot
        │  config.yaml
        │
        ├─checkpoints
        │      model_best.pth
        │
        └─log_files
                ADM-resnet12-train-Dec-08-2023-14-39-24.log


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

## 关于 5way-1shot 算法的说明

主要算法：使用 adm 作为分类器+resnet12 作为 backbone，方式是先进行预训练然后进行微调训练。

使用 LibFewShot 库中的 config/adm.yaml 文件进行预训练和微调训练。

### 预训练阶段

**特别注意：**

- 预训练使用的是修改后的 dataset.py，请复现时候千万注意

- \*\*预训练阶段的 log 文件为 1shot/log_files/\*\*ADM-resnet12-train-Dec-02-2023-13-55-09.log

**预训练的主要参数为（预训练配置为 config_pre.yaml，在 1shot 文件夹目录下）：**

```YAML
train_episode: 1000
test_episode: 1000
epoch: 50
initial_lr: 0.01
```

### 正式训练阶段

**特别注意：**

- 使用的是修改过后的 dataset.py 文件，请复现时候注意。

- 正式训练是之前预训练得到的权重文件来作为预训练模型，然后进行较小学习率的微调。

- \*\*正式训练日志文件为：1shot/log_files/ADM-resnet12-train-Dec-04-2023-23-48-27.log

- **需要在 adm.yaml(或者是 config.yaml)中加入以下内容(视具体的路径修改)：**

```YAML
pretrain_path: ./1shot/checkpoints/emb_func_best.pth
```

请注意，路径是预训练中生成的结果下 checkpoints 中的 emb_func_best.pth（为方便工作人员复现，已经将 emb_func_best.pth 文件放置在了上述路径中

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

## 关于 5way-5shot 算法的说明

算法要点：

- 使用 adm 作为分类器+resnet12 作为 backbone，方式是只进行一阶段的训练。

### 训练

**特别注意：**

- 使用的是修改过后的 dataset.py 文件，请复现时候注意

- 训练的日志文件为 5shot/log_files/ADM-resnet12-train-Dec-08-2023-14-39-24.log

**训练的主要参数为（配置文件为 config.yaml，在 5shot 文件夹目录下）：**

```YAML
train_episode: 1000
test_episode: 1000
epoch: 40
initial_lr: 0.01
augment_times: 2
test_query: 10

device_ids: 0,1
episode_size: 2
```

**训练阶段采用了数据增强，调整了 augment_times 参数为 2，同时使用了 MultiStepLR 进行动态的学习率调整，初始学习率为 0.01。**

注意 5-5 训练使用了两张 4090 显卡，所以设置了 device_ids 和 episode_size，结果可能和单卡训练出来结果的有一定差异。
