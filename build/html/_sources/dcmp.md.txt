## DCMP - Airflow集成自动生成DAG插件

Airflow 的 DAG 是通过 python 脚本来定义的，原生的 Airflow 无法通过UI界面来编辑 DAG 文件，这里介绍一个插件，通过该插件可在 UI 界面上通过拖放的方式设计工作流，最后自动生成 DAG 定义文件。

Github 地址: [Airflow DAG Creation Manager Plugin](https://github.com/lattebank/airflow-dag-creation-manager-plugin)



### 一、集成 DCMP 插件：

dcmp 插件的原理，就是在 web ui 创建完 dag 后，会根据模板 `dcmp/dag_templates/dag_code.template` 生成一段代码，然后写到本地上文件，这样就等同于我们手写的 dag 了。

#### 1. 创建 plugins 目录

```bash
mkdir  $AIRFLOW_HOME/plugins
```

#### 2. 添加

$AIRFLOW_HOME/plugins/\__init__.py

```bash
# -*- coding: utf-8 -*-
import os
import sys

from airflow import settings

PROJECT_BASE_PATH = os.path.abspath(os.path.dirname(settings.AIRFLOW_HOME))  # 项目根目录
AIRFLOW_PATH = os.path.abspath(settings.AIRFLOW_HOME)  # Airflow根目录
PLUGIN_PATH = os.path.join(AIRFLOW_PATH, "plugins")  # Plugin根目录
sys.path.extend([PROJECT_BASE_PATH, PLUGIN_PATH, AIRFLOW_PATH])
```

#### 3. 下载 dcmp 插件并移动至 plugins 下

下载地址：https://github.com/lattebank/airflow-dag-creation-manager-plugin/archive/master.zip

```bash
# 解压
unzip airflow-dag-creation-manager-plugin-master.zip

# 移动到 airflow 目录下的 plugins 下
cp -r airflow-dag-creation-manager-plugin-master $AIRFLOW_HOME/plugins
```


#### 4. 添加 airflow.cfg 配置

```bash
[webserver] 

authenticate = False
auth_backend = dcmp.auth.backends.password_auth

[dag_creation_manager]

# see https://github.com/d3/d3-3.x-api-reference/blob/master/SVG-Shapes.md#line_interpolate
# DEFAULT: basis
dag_creation_manager_line_interpolate = basis

# Choices for queue and pool
# 指定 QUEUE 和 POOL
dag_creation_manager_queue_pool = default:default|default_pool

# MR queue for queue pool
dag_creation_manager_queue_pool_mr_queue = default_pool:default

# Category for display
dag_creation_manager_category = custom

# Task category for display
dag_creation_manager_task_category = custom_task:#ffba40

# Your email address to receive email
# DEFAULT:
dag_creation_manager_default_email = your_email_address

dag_creation_manager_need_approver = False

dag_creation_manager_can_approve_self = True

dag_creation_manager_dag_templates_dir = $AIRFLOW_HOME/plugins/dcmp/dag_templates
```

关于 `dag_creation_manager_queue_pool` 的用法，可以查看源码

```python
# scheduler/plugins/dcmp/settings.py:25

DAG_CREATION_MANAGER_QUEUE_POOL = []
for queue_pool_str in DAG_CREATION_MANAGER_QUEUE_POOL_STR.split(","):
    key, queue_pool = queue_pool_str.split(":")
    queue, pool = queue_pool.split("|")
    DAG_CREATION_MANAGER_QUEUE_POOL.append((key, (queue, pool)))

DAG_CREATION_MANAGER_QUEUE_POOL_DICT = dict(DAG_CREATION_MANAGER_QUEUE_POOL)
```

如果 airflow.cfg 配置为：

```bash
# Choices for queue and pool
dag_creation_manager_queue_pool = default:master_queue|mydefault

# MR queue for queue pool
dag_creation_manager_queue_pool_mr_queue = master_queue_pool:master_mr_queue
```

以上诉为例，则`default` 为 key 的名字，这个 key 会在 dcmp 创建 dag 的 task 时显示出来，需要勾选。 `master_queue` 为 `queue` 的名字，`default_pool`为 pool 的名字。。

`airflow worker` 启动时，默认消费队列 `default`，如果这里是指定非 `default` 的，那么 worker 在启动时，需要指定对应的队列，如

```bash
airflow worker -q master_queue
```

airflow 默认的 pool 就是 `default_pool`，因此这里如果指定的 pool 不是 `default_pool`，那么就还需要在 airflow 的 web ui 中进行创建。例如上述配置的话，在 web ui 创建的 pool为 `mydefault`


#### 5. 更新数据库

```bash
python $AIRFLOW_HOME/plugins/dcmp/tools/upgradedb.py
```


### 二、启动服务

#### 1. web ui 

```bash
airflow webserver -p 8086  
```

#### 2. scheduler - 调度

```bash
airflow scheduler
```

#### 3. Worker - 消费者

```bash
# 在这里需要制定队列，跟上面 airflow.cfg 指定的队列一致
airflow worker -q mydefault
```


### 三、DCMP 下创建和管理 dag

访问：localhost:8086

如果 Admin 下出现 `DAG Creation Manager` ，就证明已经把 DCMP 集成进来了。

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/dcmp-manager.png)


#### 1. 创建 Pools

Amdin - Pool

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/admin_pool.png)

Slots 可以设置个不为0的数即可

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/create_pool.png)



#### 2. 创建 bash dag

Admin - DAG Creation Manager - Tasks

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/bash_dag.png)



保存后，我们回到 DAGs 列表查看。该DAG不会马上被识别出来，默认情况下Airflow是5分钟扫描一次dag目录，该配置可在airflow.cfg中修改。

```
[scheduler]
# dag 扫描间隔时间
dag_dir_list_interval = 10
```

过一会就看到刚刚创建的 `bash_dag_example` 了。我们可以手动触发 dag，方法是点击 `Links` 下的第一个类似暂停按钮的键即可

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/dag_list.png)

查看输出

```bash
$ tail -f /tmp/date.txt
Tue Dec  8 16:59:46 CST 2020
Tue Dec  8 17:00:06 CST 2020
Tue Dec  8 17:02:06 CST 2020
```

#### 3. 同样的方式，我们可以创建一个 python dag

类似 bash dag的创建方式，只是在 `Task` 栏勾选 `python`

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/python_dag.png)


#### 4. 查看生成的 dag 代码

选择某个 dag 并进入

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/in_python_dag.png)

查看 code 代码

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/dag_code.png)


### 四、DCMP 加入自定义的 Operator

#### 1. 如何通过 BashOperator 执行 python 命令

dag 执行的路径，不是在 dag 这个项目下，而是类似 `/private/var/folders/27/q_0dcrhx24ngz06zp7q0fck80000gn/T/airflowtmpy03npuoo`。但由于我们一般都是进入某个特定项目执行，所以如果用默认的 BashOperator，我们需要先进入某个路径

```bash
dag = DAG(
    "bash_dag_demo",
    default_args=default_args,
    description="File Caller DAG",
    schedule_interval="0 */10 * * *",
)

BashOperator(
    task_id="bash_task_demo",
    bash_command='''echo hello''',
    dag=dag,
)
```

其实对于同一个项目来说，这一步 `cd xxx` 是重复的，因此我们可以自定义的 BashOperator。只需继承默认的 BashOperator，稍作改动

scheduler/plugins/my_bash_operator 

```bash
# -*- coding: utf-8 -*-
# @Time   : 2020/12/9 上午11:31
# @Author : wu
import os

from airflow import settings
from airflow.operators.bash_operator import BashOperator
from airflow.utils.decorators import apply_defaults

PROJECT_BASE_PATH = os.path.abspath(os.path.dirname(settings.AIRFLOW_HOME))  # 项目根目录


class MyBashOperator(BashOperator):
    @apply_defaults
    def __init__(self, bash_command, xcom_push=False, env=None, output_encoding="utf-8", *args, **kwargs):
        super(BashOperator, self).__init__(*args, **kwargs)
        # bash operator 执行的路径不是当前根目录，因此当需要在通过 bash operator 执行时，还需要 cd 到项目的根目录
        # 为了减少这步操作，自定义了一个 bash_operator
        # 默认执行的路径类似：/private/var/folders/27/q_0dcrhx24ngz06zp7q0fck80000gn/T/airflowtmpy03npuoo
        self.bash_command = f"cd {PROJECT_BASE_PATH} && " + bash_command
        self.env = env
        self.xcom_push_flag = xcom_push
        self.output_encoding = output_encoding
        self.sub_process = None

```


#### 2. dcmp 加入自定的 operator

dcmp 是开源的，因此我们只需要找到相应的地方，改动下源码即可

a. dcmp/dag_templates/dag_code.template

```python
# 导入自定义的 Operator
from scheduler.plugins.my_bash_operator import MyBashOperator
```

b. dcmp/settings.py:8

```python
# 找到这句，改为如下
TASK_TYPES = ["my_bash", "bash", "hql", "python", "short_circuit", "partition_sensor", "time_sensor", "timedelta_sensor"]
```

c. dcmp/dag_creation_manager_plugin.py

```python
def command_render(task_type, command):
    attr_renderer = {
        "my_bash": lambda x: render(x, lexers.BashLexer),
        "bash": lambda x: render(x, lexers.BashLexer),
        "hql": lambda x: render(x, lexers.SqlLexer),
        "sql": lambda x: render(x, lexers.SqlLexer),
        "python": lambda x: render(x, lexers.PythonLexer),
        "short_circuit": lambda x: render(x, lexers.PythonLexer),
        "partition_sensor": lambda x: render(x, lexers.PythonLexer),
        "time_sensor": lambda x: render(x, lexers.PythonLexer),
        "timedelta_sensor": lambda x: render(x, lexers.PythonLexer),
    }
```

d. dcmp/dag_converter.py

```python
class DAGConverter(object):
    My_BASH_TASK_CODE_TEMPLATE = BASE_TASK_CODE_TEMPLATE % {
        "before_code": "",
        "operator_name": "MyBashOperator",
        "operator_code": r"""
        bash_command=r'''%(processed_command)s ''',
    """,
    }
     TASK_TYPE_TO_TEMPLATE = {
        "my_bash": My_BASH_TASK_CODE_TEMPLATE,
        "bash": BASH_TASK_CODE_TEMPLATE,
        "dummy": DUMMY_TASK_CODE_TEMPLATE,
        "hql": HQL_TASK_CODE_TEMPLATE,
        "python": PYTHON_TASK_CODE_TEMPLATE,
        "short_circuit": SHORT_CIRCUIT_TASK_CODE_TEMPLATE,
        "partition_sensor": HIVE_PARTITION_SENSOR_TASK_CODE_TEMPLATE,
        "time_sensor": TIME_SENSOR_TASK_CODE_TEMPLATE,
        "timedelta_sensor": TIMEDELTA_SENSOR_TASK_CODE_TEMPLATE,
    }

```

e.  dcmp/static/dcmp/js/edit.js

```js
window.default_task = {
        task_name: "",
        // 把自定义的改为默认
        task_type: "my_bash",
        command: "",
        priority_weight: 0,
        upstreams: []

else if(task_type == "bash"){
                    field_html.push('<textarea ' + (readonly? ' readonly="readonly" ': '') + ' id="ace_' + task_id + '_' + task_type + '" class="form-control" rows="1" name="command">' + (task_type == task[field_name]? task["command"]: '') + '</textarea>');
                    field_html.push(render_help);
                    field_html.push(get_ace_script(task_type, "sh", 1));
                }
else if(task_type == "my_bash"){
                    field_html.push('<textarea ' + (readonly? ' readonly="readonly" ': '') + ' id="ace_' + task_id + '_' + task_type + '" class="form-control" rows="1" name="command">' + (task_type == task[field_name]? task["command"]: '') + '</textarea>');
                    field_html.push(render_help);
                    field_html.push(get_ace_script(task_type, "sh", 1));
                }
```

到此，就在 dcmp 里集成了自定义的 operator 了。

### 五、DCMP 跳过非最新 dag

#### 如何跳过非最新 dag？

假设场景是这样的：我们想停掉某个 dag，有需要的时候再启动。但是当再次启动时发现，dag 会把执行停掉这段时间的任务，而不是从当前时间执行最新的任务。这明显不符合我们的需求。

如果是手写 dag 的话，需要自己写一段逻辑来跳过非最新的 dag。而 dcmp 帮我们继承了，只需要在创建 dag 的，点击 `EXpand All /Collapse All` ，然后勾选 `Skip DAG Not Latest`（跳过非最新的 dag）  和 `Skip DAGS On Prew Running`

- **Skip DAG Not Latest**：跳过非最新的 dag。如我们停一段时间后再启动，之前的任务不会执行，只会执行最新的。
- **Skip DAGS On Prew Running**：同一个 dag 的上次任务还在执行未结束，则跳过此次执行。这种适用于任务的执行的时间可能会超过任务的调度间隔时间，而同一时间我们又不想执行多个任务。

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/skip_dag_not_latest.png)



但问题来了，勾选创建 dag 之后，我们发现 `scheduler` 抛出异常了

```bash
scheduler    |   File "/usr/local/lib/python3.7/site-packages/pymysql/connections.py", line 684, in _read_packet
scheduler    |     packet.check_error()
scheduler    |   File "/usr/local/lib/python3.7/site-packages/pymysql/protocol.py", line 220, in check_error
scheduler    |     err.raise_mysql_exception(self._data)
scheduler    |   File "/usr/local/lib/python3.7/site-packages/pymysql/err.py", line 109, in raise_mysql_exception
scheduler    |     raise errorclass(errno, errval)
scheduler    | pymysql.err.IntegrityError: (1048, "Column 'pool' cannot be null")
```

说的是 pool 为空值，这时候打开 dag ，查看代码，发现 `skip_dag_not_latest_or_when_previous_running` 这个 task 下 的 `pool` 确实是 None 的。

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/pool_is_none.png)

这里的 task_name 的命名规则可看源码  scheduler/plugins/dcmp/dag_converter.py:331

```python
if conf["skip_dag_not_latest"] or conf["skip_dag_on_prev_running"]:
    task_name = []
    if conf["skip_dag_not_latest"]:
        task_name.append("not_latest")
    if conf["skip_dag_on_prev_running"]:
        task_name.append("when_previous_running")
    task_name = "_or_".join(task_name)
    task_name = "skip_dag_" + task_name
    task_name = get_task_name(task_name)
    if task_name:
        command = """
```

dag中的每个节点都是一个任务，dag中的边表示的是任务之间的依赖（强制为有向无环，因此不会出现循环依赖，从而导致无限执行循环）。

从上述 dmcp 生成的代码可以看出，当我们勾选  `Skip DAG Not Latest`  或 `Skip DAGS On Prew Running` 时，其他会生成一个 task，且会与我们原本创建的 task（在上图是 task_test）形成一个依赖关系，A << B 的意思是 A 依赖于 B，B先执行A后执行。

| 序号 | 概念                | 解释                     |
| :--- | :------------------ | :----------------------- |
| 1    | A >> B              | B依赖于A，A先执行B后执行 |
| 2    | A << B              | A依赖于B，B先执行A后执行 |
| 3    | A.set_downstream(B) | 等同于 A >> B            |
| 4    | A.set_upstream(B)   | 等同于 B >> A            |

### 修复源码 bug

从源码看起来这应该是个 bug？因此当我们通过勾选  `Skip DAG Not Latest` 这种方式生成的 task，是没有指定 **Queue Pool**的，因此这里的 queue pool 为 None ，此时 `queue_code`(即 queue) 取 `configuration.get("celery", "default_queue")` 默认值为 `default`，`pool_code = None`

那我们从源码入手，修改源代码: dcmp/dag_converter.py:432

```python
def render_confs(self, confs):
    for task in conf["tasks"]:
        queue_pool = dcmp_settings.DAG_CREATION_MANAGER_QUEUE_POOL_DICT.get(task["queue_pool"])
        if queue_pool:
            queue, pool = queue_pool
            task["queue_code"] = "'%s'" % queue
            task["pool_code"] = "'%s'" % pool
        else:
            task["queue_code"] = "'%s'" % configuration.get("celery", "default_queue")
            task["pool_code"] = "None"
```

知道这个问题后，那我们来修复下就好了

```python
def render_confs(self, confs):
    for task in conf["tasks"]:
        queue_pool = dcmp_settings.DAG_CREATION_MANAGER_QUEUE_POOL_DICT.get(task["queue_pool"])
        if not queue_pool:
            # 获取默认的第一个配置
            queue_pool = dcmp_settings.DAG_CREATION_MANAGER_QUEUE_POOL[0][-1]
        if queue_pool:
            queue, pool = queue_pool
            task["queue_code"] = "'%s'" % queue
            task["pool_code"] = "'%s'" % pool
```

这样就是获取了 airflow.cfg 下的第一个配置

```bash
dag_creation_manager_queue_pool = default:default|default_pool,my_test:test_queue|test_pool
```

如上，如果配置了两个，就是默认获取 `default` 这个，其 queue 为 `default`，pool 为 `default_pool`。

当然，以上源码的改动是按照我自己的需求来改的，如果有其他需求，就按其他方式改，这里仅提供一个参考。

修改完后，再创建的 dag 中，`skip_dag_not_latest_or_when_previous_running` 应该如下。

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/airflow/pool_is_not_none.png)

源码默认`Skip DAG Not Latest`  或 `Skip DAGS On Prew Running` 都为 `False`。

但如果我们的场景都应该为 True，每次都去勾选有点麻烦，这时候我们还可以改下源码 dcmp/dag_creation_manager_plugin.py
在 DEFAULT_CONF 配置下将 `skip_dag_not_latest` 和 `skip_dag_on_prev_running` 设置为 True
```python
DEFAULT_CONF = {
        "skip_dag_not_latest": True,
        "skip_dag_on_prev_running": True,
    }
```