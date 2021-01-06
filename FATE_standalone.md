# FATE

该文档为学习研究、部署测试、以及开发FATE项目的内容记录。（版本号：1.5）

FATE (Federated AI Technology Enabler),微众银行AI部门发起的开源项目。[Github地址](https://github.com/FederatedAI/FATE)

- [待完成内容](#待完成内容)
- [基础结构与算法](#基础结构与算法)
  - [离线部分](#离线部分)
  - [在线部分](#在线部分)
- [部署](#部署)
  - [单机部署](#单机部署)
- [运行测试](#运行测试)

## 待完成内容

目前待回答问题、待完成内容有：

1. 是否可以直接生成dsl与conf配置文档？
2. 如何获取模型的evaluation metric？
3. 安全算法

## 基础结构与算法

在总体架构上，FATE架构分为离线部分与在线部分。其中，离线部分主要负责模型的训练，而在线部分负责进行推理。

### 离线部分

作为联盟模型的模型训练部分，主要模块有fate flow, federate dml, fate board。

- fate flow: 联邦学习模块运行和管理，主要负责提交任务，解析参数，生成作业，保存/查询日志。

- federate dml: 联邦学习的主要实现模块。

- fate board: 模型可视化页面。

其他支持部分：

- poxy: 网络对外接口

- redis: 为fate flow提供队列支持

- eggroll: 为重开发的分布式数据存储/计算框架。（FATE 支持从 eggroll数据库和 spark 数据库调用数据）。

### 在线部分

- Serving Server：主要负责模型加载和缓存以及在线推理。
- Serving Poxy：在线部分的网络出口。可对接业务出口。
- Zookeeper：在线部分服务发现与管理。

## 部署

### 单机部署

1. docker部署下载、添加镜像：

  ```bash
  # 下载
  wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/docker_standalone-fate-1.5.0.tar.gz

  # 解压
  tar -xzvf docker_standalone-fate-1.5.0.tar.gz

  # 拉取镜像
  bash install_standalone_docker.sh

  # 确认docker镜像加载成功
  docker ps

  # 创建容器
  CONTAINER_ID=`docker ps -aqf "name=fate_python"`

  # 进入容器
  docker exec -t -i fate_python bash
  ```

2. 上传数据到eggroll管理器

    2.1 配置json文件（例）：

    ```json
    {
      "file": "${path to csv file}",
      "id_delimiter":",",
      "head": 1,
      // 忽略header: 0
      // 读取header: 1
      "partition": 4,
      "work_mode": 0,
      // 单机部署模式: 0 
      // 集群部署模式: 1
      "backend":0,
      // ????
      "namespace": "${namespace}",
      "table_name": "${table_name}"
    }
    ```

    2.2 提交工作请求

    ```bash
    python ${your_install_path}/fate_flow/fate_flow_client.py -f upload -c ${config_json_file} -c 
    ```

    2.3 获取上传结果

    ```bash
    {
    "data": {
        "board_url": "http://0.0.0.0:8080/index.html#/dashboard?job_id=2021010609090551568921&role=local&party_id=0",
        "job_dsl_path": "/fate/jobs/2021010609090551568921/job_dsl.json",
        "job_id": "2021010609090551568921",
        "job_runtime_conf_on_party_path": "/fate/jobs/2021010609090551568921/local/job_runtime_on_party_conf.json",
        "job_runtime_conf_path": "/fate/jobs/2021010609090551568921/job_runtime_conf.json",
        "logs_directory": "/fate/logs/2021010609090551568921",
        "model_info": {
            "model_id": "local-0#model",
            "model_version": "2021010609090551568921"
        },
        "namespace": "experiment",
        "pipeline_dsl_path": "/fate/jobs/2021010609090551568921/pipeline_dsl.json",
        "table_name": "breast_guest_train",
        "train_runtime_conf_path": "/fate/jobs/2021010609090551568921/train_runtime_conf.json"
      },
    "jobId": "2021010609090551568921",
    "retcode": 0,
    "retmsg": "success"
    }
    
    ```

3. 上传训练模型

    3.1 配置dsl文件：

    ```json
    {
        "components": {
            "dataio_0": {
                "module": "DataIO",
                "input": {
                    "data": {
                        "data": [
                            "args.train_data"
                        ]
                    }
                },
                "output": {
                    "data": [
                        "train"
                    ],
                    "model": [
                        "dataio"
                    ]
                }
            },
            "dataio_1": {
                "module": "DataIO",
                "input": {
                    "data": {
                        "data": [
                            "args.eval_data"
                        ]
                    },
                    "model": [
                        "dataio_0.dataio"
                    ]
                },
                "output": {
                    "data": [
                        "eval"
                    ],
                    "model": [
                        "dataio"
                    ]
                },
                "need_deploy": false
            },
            "intersection_0": {
                "module": "Intersection",
                "input": {
                    "data": {
                        "data": [
                            "dataio_0.train"
                        ]
                    }
                },
                "output": {
                    "data": [
                        "train"
                    ]
                }
            },
            "intersection_1": {
                "module": "Intersection",
                "input": {
                    "data": {
                        "data": [
                            "dataio_1.eval"
                        ]
                    }
                },
                "output": {
                    "data": [
                        "eval"
                    ]
                },
                "need_deploy": false
            },
            "secureboost_0": {
                "module": "HeteroSecureBoost",
                "input": {
                    "data": {
                        "train_data": [
                            "intersection_0.train"
                        ],
                        "eval_data": [
                            "intersection_1.eval"
                        ]
                    }
                },
                "output": {
                    "data": [
                        "train"
                    ],
                    "model": [
                        "train"
                    ]
                }
            },
            "evaluation_0": {
                "module": "Evaluation",
                "input": {
                    "data": {
                        "data": [
                            "secureboost_0.train"
                        ]
                    }
                }
            }
        }
    }
    ```

    3.2 配置conf文件

    ```json
    {
        "initiator": {
            "role": "guest",
            "party_id": 10000
        },
        "job_parameters": {
            "work_mode": 0
        },
        "role": {
            "guest": [
                10000
            ],
            "host": [
                10000
            ]
        },
        "role_parameters": {
            "guest": {
                "args": {
                    "data": {
                        "train_data": [
                            {
                                "name": "breast_guest_train",
                                "namespace": "experiment"
                            }
                        ],
                        "eval_data": [
                            {
                                "name": "breast_guest_train",
                                "namespace": "experiment"
                            }
                        ]
                    }
                },
                "dataio_0": {
                    "with_label": [
                        true
                    ],
                    "label_name": [
                        "y"
                    ],
                    "label_type": [
                        "int"
                    ],
                    "output_format": [
                        "dense"
                    ]
                }
            },
            "host": {
                "args": {
                    "data": {
                        "train_data": [
                            {
                                "name": "breast_host_train",
                                "namespace": "experiment"
                            }
                        ],
                        "eval_data": [
                            {
                                "name": "breast_host_train",
                                "namespace": "experiment"
                            }
                        ]
                    }
                },
                "dataio_0": {
                    "with_label": [
                        false
                    ],
                    "output_format": [
                        "dense"
                    ]
                }
            }
        },
        "algorithm_parameters": {
            "secureboost_0": {
                "task_type": "classification",
                "learning_rate": 0.1,
                "num_trees": 3,
                "subsample_feature_rate": 1,
                "n_iter_no_change": false,
                "tol": 0.0001,
                "bin_num": 50,
                "tree_param": {
                    "max_depth": 3
                },
                "objective_param": {
                    "objective": "cross_entropy"
                },
                "encrypt_param": {
                    "method": "iterativeAffine"
                },
                "predict_param": {
                    "threshold": 0.5
                },
                "cv_param": {
                    "n_splits": 5,
                    "shuffle": false,
                    "random_seed": 103,
                    "need_cv": false
                },
                "validation_freqs": 1
            },
            "evaluation_0": {
                "eval_type": "binary"
            }
        }
    }
    ```
  
    3.3 提交模型

    ```bash
    python ${your_install_path}/fate_flow/fate_flow_client.py -f submit_job -d ${your_dsl.json} -c ${your_job_conf.json}
    ```

4. 测试并输出测试结果
    **待续**
