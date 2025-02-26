# 在标注数据集上测试和训练

### 在标准数据集上测试已有模型

- 单显卡
- CPU
- 单节点多显卡
- 多节点

你可以通过以下命令来测试数据集：

```shell
# 单块显卡测试
python tools/test.py ${CONFIG_FILE} ${CHECKPOINT_FILE} [--out ${RESULT_FILE}] [--eval ${EVAL_METRICS}] [--show] [--show-dir ${SHOW_DIR}]

# CPU：禁用显卡并运行单块 CPU 测试脚本（实验性）
export CUDA_VISIBLE_DEVICES=-1
python tools/test.py ${CONFIG_FILE} ${CHECKPOINT_FILE} [--out ${RESULT_FILE}] [--eval ${EVAL_METRICS}] [--show] [--show-dir ${SHOW_DIR}]

# 多块显卡测试
./tools/dist_test.sh ${CONFIG_FILE} ${CHECKPOINT_FILE} ${GPU_NUM} [--out ${RESULT_FILE}] [--eval ${EVAL_METRICS}]
```

**注意**:

目前我们只支持 SMOKE 的 CPU 推理测试。

可选参数：

- `--show`：如果被指定，检测结果会在静默模式下被保存，用于调试和可视化，但只在单块 GPU 测试的情况下生效，和 `--show-dir` 搭配使用。
- `--show-dir`：如果被指定，检测结果会被保存在指定文件夹下的 `***_points.obj` 和 `***_pred.obj` 文件中，用于调试和可视化，但只在单块 GPU 测试的情况下生效，对于这个选项，图形化界面在你的环境中不是必需的。

所有和评估相关的参数在相应的数据集配置的 `test_evaluator` 中设置。例如 `test_evaluator = dict(type='KittiMetric', ann_file=data_root + 'kitti_infos_val.pkl', pklfile_prefix=None, submission_prefix=None)`

参数：

- `type`：相对应的评价指标名，通常和数据集相关联。
- `ann_file`：标注文件路径。
- `pklfile_prefix`：可选参数。输出结果保存成 pickle 格式的文件名。如果没有指定，结果将不会保存成文件。
- `submission_prefix`：可选参数。结果将被保存到文件中，然后你可以将它上传到官方评估服务器中。

示例：

假定你已经把模型权重文件下载到 `checkpoints/` 文件夹下，

1. 在 ScanNet 数据集上测试 VoteNet，保存模型，可视化预测结果

   ```shell
   python tools/test.py configs/votenet/votenet_8xb8_scannet-3d.py \
       checkpoints/votenet_8x8_scannet-3d-18class_20200620_230238-2cea9c3a.pth \
       --show --show-dir ./data/scannet/show_results
   ```

2. 在 ScanNet 数据集上测试 VoteNet，保存模型，可视化预测结果，可视化真实标签，计算 mAP

   ```shell
   python tools/test.py configs/votenet/votenet_8xb8_scannet-3d.py \
       checkpoints/votenet_8x8_scannet-3d-18class_20200620_230238-2cea9c3a.pth \
       --show --show-dir ./data/scannet/show_results
   ```

3. 在 ScanNet 数据集上测试 VoteNet（不保存测试结果），计算 mAP

   ```shell
   python tools/test.py configs/votenet/votenet_8xb8_scannet-3d.py \
       checkpoints/votenet_8x8_scannet-3d-18class_20200620_230238-2cea9c3a.pth
   ```

4. 使用 8 块显卡在 KITTI 数据集上测试 SECOND，计算 mAP

   ```shell
   ./tools/slurm_test.sh ${PARTITION} ${JOB_NAME} configs/second/second_hv_secfpn_8xb6-80e_kitti-3d-3class.py \
       checkpoints/hv_second_secfpn_6x8_80e_kitti-3d-3class_20200620_230238-9208083a.pth
   ```

5. 使用 8 块显卡在 nuScenes 数据集上测试 PointPillars，生成提交给官方评测服务器的 json 文件

   ```shell
   ./tools/slurm_test.sh ${PARTITION} ${JOB_NAME} configs/pointpillars/pointpillars_hv_secfpn_sbn-all_8xb4-2x_nus-3d.py \
       checkpoints/hv_pointpillars_fpn_sbn-all_4x8_2x_nus-3d_20200620_230405-2fa62f3d.pth \
      --cfg-options 'test_evaluator.jsonfile_prefix=./pointpillars_nuscenes_results'
   ```

   生成的结果会保存在 `./pointpillars_nuscenes_results` 目录。

6. 使用 8 块显卡在 KITTI 数据集上测试 SECOND，生成提交给官方评测服务器的 txt 文件

   ```shell
   ./tools/slurm_test.sh ${PARTITION} ${JOB_NAME} configs/second/second_hv_secfpn_8xb6-80e_kitti-3d-3class.py \
       checkpoints/hv_second_secfpn_6x8_80e_kitti-3d-3class_20200620_230238-9208083a.pth \
       --cfg-options 'test_evaluator.pklfile_prefix=./second_kitti_results' 'test_evaluator.submission_prefix=./second_kitti_results'
   ```

   生成的结果会保存在 `./second_kitti_results` 目录。

7. 使用 8 块显卡在 Lyft 数据集上测试 PointPillars，生成提交给排行榜的 pkl 文件

   ```shell
   ./tools/slurm_test.sh ${PARTITION} ${JOB_NAME} configs/pointpillars/hv_pointpillars_fpn_sbn-2x8_2x_lyft-3d.py \
       checkpoints/hv_pointpillars_fpn_sbn-2x8_2x_lyft-3d_latest.pth \
       --cfg-options 'test_evaluator.jsonfile_prefix=results/pp_lyft/results_challenge' \
       'test_evaluator.csv_savepath=results/pp_lyft/results_challenge.csv' \
       'test_evaluator.pklfile_prefix=results/pp_lyft/results_challenge.pkl'
   ```

   **注意**：为了生成 Lyft 数据集的提交结果，`--eval-options` 必须指定 `csv_savepath`。生成 csv 文件后，你可以使用[网站](https://www.kaggle.com/c/3d-object-detection-for-autonomous-vehicles/submit)上给出的 kaggle 命令提交结果。

   注意在 [Lyft 数据集的配置文件](../../configs/_base_/datasets/lyft-3d.py)，`test` 中的 `ann_file` 值为 `lyft_infos_test.pkl`，是没有标注的 Lyft 官方测试集。要在验证数据集上测试，请把它改为 `lyft_infos_val.pkl`。

8. 使用 8 块显卡在 waymo 数据集上测试 PointPillars，使用 waymo 度量方法计算 mAP

   ```shell
   ./tools/slurm_test.sh ${PARTITION} ${JOB_NAME} configs/pointpillars/pointpillars_hv_secfpn_sbn-all_16xb2-2x_waymo-3d-car.py  \
       checkpoints/hv_pointpillars_secfpn_sbn-2x16_2x_waymo-3d-car_latest.pth \
       --cfg-options 'test_evaluator.result_prefix=results/waymo-car/kitti_results' \
       'test_evaluator.format_only=True'
   ```

   **注意**：对于 waymo 数据集上的评估，请根据[说明](https://github.com/waymo-research/waymo-open-dataset/blob/v1.5.0/docs/quick_start.md)或[教程](https://github.com/waymo-research/waymo-open-dataset/blob/v1.5.0/tutorial/tutorial.ipynb)构建二进制文件 `compute_detection_metrics_main` 来做度量计算，并把它放在本工程 `mmdet3d/evaluation/functional/waymo_utils/`下。（在使用 bazel 构建  `compute_detection_metrics_main` 时，有时会出现 `'round' is not a member of 'std'` 的错误，我们只需要把那个文件中 `round` 前的 `std::` 去掉。）二进制文件生成时需要在 `--eval-options` 中给定 `pklfile_prefix`。对于度量方法，`waymo` 是推荐的官方评估策略，目前 `kitti` 评估是依照 KITTI 而来的，每个难度的结果和 KITTI 的定义并不完全一致。目前大多数物体都被标记为0难度，会在未来修复。它的不稳定原因包括评估的计算大、转换后的数据缺乏遮挡和截断、难度的定义不同以及平均精度的计算方法不同。

9. 使用 8 块显卡在 waymo 数据集上测试 PointPillars，生成 bin 文件并提交到排行榜

   ```shell
   ./tools/slurm_test.sh ${PARTITION} ${JOB_NAME} configs/pointpillars/pointpillars_hv_secfpn_sbn-all_16xb2-2x_waymo-3d-car.py  \
       checkpoints/hv_pointpillars_secfpn_sbn-2x16_2x_waymo-3d-car_latest.pth \
       --cfg-options 'test_evaluator.result_prefix=results/waymo-car/kitti_results' \
       'test_evaluator.format_only=True'
   ```

   **注意**：生成 bin 文件后，你可以简单地构建二进制文件  `create_submission`，并根据[说明](https://github.com/waymo-research/waymo-open-dataset/blob/v1.5.0/docs/quick_start.md)创建提交的文件。要在验证服务器上评测验证数据集，你也可以用同样的方式生成提交的文件。

## 在标准数据集上训练预定义模型

MMDetection3D 分别用 `MMDistributedDataParallel` and `MMDataParallel` 实现了分布式训练和非分布式训练。

所有的输出（日志文件和模型权重文件）都会被保存到工作目录下，通过配置文件里的 `work_dir` 指定。

默认我们每过一个周期都在验证数据集上评测模型，你可以通过在训练配置里添加间隔参数来改变评测的时间间隔：

```python
train_cfg = dict(type='EpochBasedTrainLoop', val_interval=1)  # 每12个周期评估一次模型
```

**重要**：配置文件中的默认学习率对应 8 块显卡，配置文件名里有具体的批量大小，比如 '2xb8' 表示一共 8 块显卡，每块显卡 2 个样本。
根据 [Linear Scaling Rule](https://arxiv.org/abs/1706.02677)，当你使用不同数量的显卡或每块显卡有不同数量的图像时，需要依批量大小按比例调整学习率。如果用 4 块显卡、每块显卡 2 幅图像时学习率为 0.01，那么用 16 块显卡、每块显卡 4 幅图像时学习率应设为 0.08。然而，由于大多数模型使用 ADAM 而不是 SGD 进行优化，上述规则可能并不适用，用户需要自己调整学习率。

### 使用单块显卡进行训练

```shell
python tools/train.py ${CONFIG_FILE} [optional arguments]
```

如果你想在命令中指定工作目录，添加参数 `--work-dir ${YOUR_WORK_DIR}`。

### 使用 CPU 进行训练 (实验性)

在 CPU 上训练的过程与单 GPU 训练一致。 我们只需要在训练过程之前禁用显卡。

```shell
export CUDA_VISIBLE_DEVICES=-1
```

之后运行单显卡训练脚本即可。

**注意**：

目前，大多数点云相关算法都依赖于 3D CUDA 算子，无法在 CPU 上进行训练。 一些单目 3D 物体检测算法，例如 FCOS3D、SMOKE 可以在 CPU 上进行训练。我们不推荐用户使用 CPU 进行训练，这太过缓慢。我们支持这个功能是为了方便用户在没有显卡的机器上调试某些特定的方法。

### 使用多块显卡进行训练

```shell
./tools/dist_train.sh ${CONFIG_FILE} ${GPU_NUM} [optional arguments]
```

可选参数：

- `--cfg-options 'Key=value'`：覆盖使用的配置中的一些设定。

### 使用多个机器进行训练

如果要在 [slurm](https://slurm.schedmd.com/) 管理的集群上运行 MMDectection3D，你可以使用 `slurm_train.sh` 脚本（该脚本也支持单机训练）

```shell
[GPUS=${GPUS}] ./tools/slurm_train.sh ${PARTITION} ${JOB_NAME} ${CONFIG_FILE} ${WORK_DIR}
```

下面是一个使用 16 块显卡在 dev 分区上训练 Mask R-CNN 的示例：

```shell
GPUS=16 ./tools/slurm_train.sh dev pp_kitti_3class configs/pointpillars/pointpillars_hv_secfpn_8xb6-160e_kitti-3d-3class.py /nfs/xxxx/pp_kitti_3class
```

你可以查看 [slurm_train.sh](https://github.com/open-mmlab/mmdetection/blob/master/tools/slurm_train.sh) 来获取所有的参数和环境变量。

如果您想使用由 ethernet 连接起来的多台机器， 您可以使用以下命令:

在第一台机器上:

```shell
NNODES=2 NODE_RANK=0 PORT=$MASTER_PORT MASTER_ADDR=$MASTER_ADDR ./tools/dist_train.sh $CONFIG $GPUS
```

在第二台机器上:

```shell
NNODES=2 NODE_RANK=1 PORT=$MASTER_PORT MASTER_ADDR=$MASTER_ADDR ./tools/dist_train.sh $CONFIG $GPUS
```

但是，如果您不使用高速网路连接这几台机器的话，训练将会非常慢。

### 在单个机器上启动多个任务

如果你在单个机器上启动多个任务，比如，在具有8块显卡的机器上进行2个4块显卡训练的任务，你需要为每个任务指定不同的端口（默认为29500）以避免通信冲突。

如果你使用 `dist_train.sh` 启动训练任务，可以在命令中设置端口：

```shell
CUDA_VISIBLE_DEVICES=0,1,2,3 PORT=29500 ./tools/dist_train.sh ${CONFIG_FILE} 4
CUDA_VISIBLE_DEVICES=4,5,6,7 PORT=29501 ./tools/dist_train.sh ${CONFIG_FILE} 4
```

如果你使用 Slurm 启动训练任务，有两种方式指定端口：

1. 通过 `--cfg-options` 设置端口，这是更推荐的，因为它不改变原来的配置

   ```shell
   CUDA_VISIBLE_DEVICES=0,1,2,3 GPUS=4 ./tools/slurm_train.sh ${PARTITION} ${JOB_NAME} config1.py ${WORK_DIR} --cfg-options 'env_cfg.dist_cfg.port=29500'
   CUDA_VISIBLE_DEVICES=4,5,6,7 GPUS=4 ./tools/slurm_train.sh ${PARTITION} ${JOB_NAME} config2.py ${WORK_DIR} --cfg-options 'env_cfg.dist_cfg.port=29501'
   ```

2. 修改配置文件（通常在配置文件的倒数第6行）来设置不同的通信端口

   在 `config1.py` 中，

   ```python
   env_cfg = dict(
       dist_cfg=dict(backend='nccl', port=29500)
   )
   ```

   在 `config2.py` 中，

   ```python
   env_cfg = dict(
       dist_cfg=dict(backend='nccl', port=29501)
   )
   ```

   然后，你可以使用 `config1.py` and `config2.py` 启动两个任务

   ```shell
   CUDA_VISIBLE_DEVICES=0,1,2,3 GPUS=4 ./tools/slurm_train.sh ${PARTITION} ${JOB_NAME} config1.py ${WORK_DIR}
   CUDA_VISIBLE_DEVICES=4,5,6,7 GPUS=4 ./tools/slurm_train.sh ${PARTITION} ${JOB_NAME} config2.py ${WORK_DIR}
   ```
