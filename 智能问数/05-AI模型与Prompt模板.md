# 05 - AI 模型与 Prompt 模板

> **阅读目标**：理解 SQLBot 如何管理和调用不同的 LLM，以及 Prompt 模板系统的设计

---

## 1. AI 模型管理

### 1.1 数据模型

AI 模型配置存储在 `ai_model_detail` 表中：

```python
class AiModelDetail(SQLModel, table=True):
    __tablename__ = "ai_model_detail"
    id: int
    name: str              # 模型显示名称
    base_model: str        # 模型标识（如 gpt-4, qwen-turbo）
    protocol: int          # 协议类型：1=OpenAI兼容, 2=vLLM
    api_domain: str        # API 地址
    api_key: str           # API 密钥（加密存储）
    config: str            # 额外配置（JSON，如 temperature）
    default_model: bool    # 是否为默认模型
    description: str       # 描述
    oid: int               # 所属工作空间（NULL=全局）
```

### 1.2 模型工厂模式

`LLMFactory` 使用**工厂模式**创建不同类型的 LLM 实例：

```python
# apps/ai_model/model_factory.py

class LLMFactory:
    _llm_types = {
        "openai": OpenAILLM,      # OpenAI / 兼容 API
        "tongyi": OpenAILLM,      # 通义千问（也用 OpenAI 兼容格式）
        "vllm": OpenAIvLLM,       # vLLM 推理引擎
        "azure": OpenAIAzureLLM,  # Azure OpenAI
    }

    @classmethod
    @lru_cache(maxsize=32)  # 缓存 LLM 实例，避免重复创建
    def create_llm(cls, config: LLMConfig) -> BaseLLM:
        llm_class = cls._llm_types.get(config.model_type)
        return llm_class(config)
```

> 💡 `@lru_cache(maxsize=32)` 确保同一个模型配置不会重复创建 LLM 实例，最多缓存 32 个。

### 1.3 LLM 类层次结构

```
BaseLLM (抽象基类)
├── OpenAILLM        → BaseChatOpenAI (自定义封装，兼容 OpenAI API)
├── OpenAIvLLM       → VLLMOpenAI (vLLM 推理引擎)
└── OpenAIAzureLLM   → AzureChatOpenAI (Azure OpenAI)
```

每个子类只需实现 `_init_llm()` 方法：

```python
class OpenAILLM(BaseLLM):
    def _init_llm(self) -> BaseChatModel:
        return BaseChatOpenAI(
            model=self.config.model_name,
            api_key=self.config.api_key or 'Empty',
            base_url=self.config.api_base_url,
            stream_usage=True,       # 启用流式输出
            **self.config.additional_params,  # temperature 等参数
        )
```

### 1.4 LLMConfig — 模型配置

```python
class LLMConfig(BaseModel):
    model_id: int          # 数据库中的模型 ID
    model_type: str        # "openai" / "vllm" / "azure"
    model_name: str        # 模型名称
    api_key: str           # API 密钥
    api_base_url: str      # API 地址
    additional_params: dict # 额外参数 {"temperature": 0.7, "max_tokens": 4096, ...}
```

### 1.5 获取默认模型配置

```python
async def get_default_config(custom_model_id=None) -> LLMConfig:
    with Session(engine) as session:
        # 如果指定了模型 ID，用指定的
        if custom_model_id:
            db_model = session.get(AiModelDetail, custom_model_id)

        # 否则用默认模型
        if not db_model:
            db_model = session.exec(
                select(AiModelDetail).where(AiModelDetail.default_model == True)
            ).first()

        # 解密 API 密钥和地址
        if not db_model.api_domain.startswith("http"):
            db_model.api_domain = await sqlbot_decrypt(db_model.api_domain)
            db_model.api_key = await sqlbot_decrypt(db_model.api_key)

        return LLMConfig(
            model_id=db_model.id,
            model_type="openai" if db_model.protocol == 1 else "vllm",
            model_name=db_model.base_model,
            api_key=db_model.api_key,
            api_base_url=db_model.api_domain,
            additional_params=additional_params,
        )
```

---

## 2. Prompt 模板系统

### 2.1 模板目录结构

```
backend/
├── apps/template/           # Prompt 模板生成器（Python 代码）
│   ├── template.py          # YAML 模板加载器
│   ├── generate_sql/        # SQL 生成模板
│   │   └── generator.py     #   get_sql_template() → 返回 system/user prompt
│   ├── generate_chart/      # 图表生成模板
│   │   └── generator.py
│   ├── generate_analysis/   # 数据分析模板
│   │   └── generator.py
│   ├── generate_predict/    # 数据预测模板
│   │   └── generator.py
│   ├── generate_dynamic/    # 动态 SQL 模板
│   │   └── generator.py
│   ├── generate_guess_question/ # 猜你想问模板
│   │   └── generator.py
│   ├── select_datasource/   # 数据源选择模板
│   │   └── generator.py
│   └── filter/              # 权限过滤模板
│       └── generator.py
│
└── templates/               # YAML 模板文件（实际的 Prompt 内容）
    ├── template.yaml        # 基础模板（通用的系统提示词）
    └── sql_examples/        # 各数据库的 SQL 示例
        ├── mysql.yaml
        ├── postgresql.yaml
        ├── oracle.yaml
        └── ...
```

### 2.2 模板加载机制

```python
# apps/template/template.py

@cache  # 缓存，避免重复读文件
def _load_template_file(file_path: Path):
    with open(file_path, 'r', encoding='utf-8') as f:
        return yaml.safe_load(f)

def get_base_template():
    """加载基础模板"""
    return _load_template_file(BASE_TEMPLATE_PATH)

def get_sql_template(db_type):
    """根据数据库类型加载对应的 SQL 示例模板"""
    db_enum = DB.get_db(db_type)
    template_path = SQL_TEMPLATES_DIR / f"{db_enum.template_name}.yaml"
    return _load_template_file(template_path)
```

### 2.3 Generator 模式

每个 `generator.py` 提供统一的接口，读取 YAML 模板并替换变量：

```python
# apps/template/generate_sql/generator.py

def get_sql_template():
    """返回 SQL 生成的 Prompt 模板"""
    base = get_base_template()
    sql_specific = ...  # 从 SQL 模板文件加载

    return {
        'system': base['sql']['system'],     # 系统提示词
        'rules': base['sql']['rules'],        # 规则说明
        'schema': '...',                      # 表结构（运行时填充）
        'terminologies': '...',               # 术语表（运行时填充）
        'data_training': '...',               # SQL 示例（运行时填充）
        'custom_prompt': '...',               # 自定义 Prompt（运行时填充）
    }
```

### 2.4 ChatQuestion 中的 Prompt 构建方法

`ChatQuestion` 类中定义了一系列方法，负责把各种信息拼装成最终的 Prompt：

```python
class ChatQuestion:
    def sql_sys_question(self, ds_type, enable_row_limit):
        """构建 SQL 生成的系统提示词"""
        templates = get_sql_template()
        return {
            'system': templates['system'],
            'rules': templates['rules'],
            'schema': self._build_schema_section(),
            'terminologies': self._build_terminology_section(),
            'data_training': self._build_training_section(),
            'custom_prompt': self._build_custom_prompt_section(),
        }

    def sql_user_question(self, current_time, change_title=False):
        """构建用户的实际问题"""
        return f"""
<m-question>
{self.question}
</m-question>
当前时间：{current_time}
"""

    def chart_sys_question(self):
        """构建图表生成的系统提示词"""
        ...

    def chart_user_question(self, chart_type, schema):
        """构建图表生成的用户提示词"""
        ...

    def analysis_sys_question(self):
        """构建数据分析的系统提示词"""
        ...
```

### 2.5 Prompt 中使用的特殊标签

项目中使用了自定义标签来组织 Prompt 内容：

| 标签 | 用途 | 出现位置 |
|------|------|---------|
| `<m-schema>` | 数据库表结构 | SQL 生成 Prompt |
| `<m-question>` | 用户问题 | SQL 生成 Prompt |
| `<m-sample-data>` | 表样本数据 | SQL 生成 Prompt |
| `<error-msg>` | 上次 SQL 执行错误 | SQL 生成 Prompt（纠错） |
| `<field-constraints>` | 用户确认的字段约束 | SQL 生成 Prompt |
| `<m-terminology>` | 术语表 | SQL 生成 Prompt |
| `<m-data-training>` | SQL 示例 | SQL 生成 Prompt |
| `<m-rules>` | 规则说明 | 所有 Prompt |
| `<field-constraints priority="CRITICAL">` | 强制字段约束 | SQL 生成 Prompt |

---

## 3. 各类 Prompt 的作用

### 3.1 SQL 生成 Prompt

目标：让 LLM 根据用户问题和表结构生成正确的 SQL

```
系统角色：你是一个 SQL 专家
规则：
  - 只使用提供的表和字段
  - 遵守 SQL 语法规范
  - 输出 JSON 格式：{"success": true, "sql": "...", "brief": "...", "chart-type": "..."}
  - 不要执行写操作
表结构：<m-schema>...</m-schema>
术语表：<m-terminology>...</m-terminology>
SQL 示例：<m-data-training>...</m-data-training>
用户问题：<m-question>上个月各部门的销售额</m-question>
```

### 3.2 图表生成 Prompt

目标：根据 SQL 结果生成合适的图表配置

```
根据以下数据，生成一个合适的图表配置：
数据列：{fields}
SQL：{sql}
表结构：{schema}

输出 JSON 格式：
{
  "type": "bar|line|pie|table|...",
  "title": "图表标题",
  "columns": [{"name": "显示名", "value": "字段名"}],
  "axis": {
    "x": {"name": "X轴", "value": "field"},
    "y": {"name": "Y轴", "value": "field"},
    "series": ...
  }
}
```

### 3.3 数据分析 Prompt

目标：对已有数据进行深度分析，生成文字报告

### 3.4 数据预测 Prompt

目标：基于历史数据预测未来趋势

### 3.5 数据源选择 Prompt

目标：当用户没有指定数据源时，让 LLM 根据问题选择最合适的数据库

### 3.6 权限过滤 Prompt

目标：将行级权限条件注入到已有 SQL 中

```
请根据以下权限规则，修改 SQL 添加过滤条件：
原始 SQL：{sql}
权限规则：{filters}

输出修改后的 SQL。
```

---

## 4. Embedding 模型

除了 LLM，SQLBot 还使用 Embedding 模型进行语义匹配：

```python
# apps/ai_model/embedding.py

class EmbeddingModelCache:
    """Embedding 模型缓存"""

    @classmethod
    def get_model(cls):
        """获取默认 Embedding 模型"""
        # 默认使用 shibing624/text2vec-base-chinese
        # 这是一个中文文本向量模型
        ...
```

Embedding 模型用于：
- 匹配相关术语表
- 匹配 SQL 训练示例
- 匹配相关数据源
- 匹配相关表结构

详见 [06-向量嵌入与语义匹配](./06-向量嵌入与语义匹配.md)

---

## 小结

1. **模型工厂**：`LLMFactory` 通过工厂模式 + 缓存创建不同类型的 LLM 实例
2. **模板系统**：YAML 文件存储 Prompt 模板内容，Python 代码负责变量替换和组装
3. **Prompt 结构**：使用自定义标签（`<m-schema>`, `<m-question>` 等）组织内容
4. **多种 Prompt**：SQL 生成、图表生成、分析、预测、数据源选择、权限过滤各有独立的 Prompt
5. **双模型架构**：LLM（文本生成）+ Embedding（语义匹配）协同工作

---

[← 返回总览](./00-总览.md) | [上一篇：AI 模型与 Prompt 模板](./05-AI模型与Prompt模板.md) | [下一篇：向量嵌入与语义匹配 →](./06-向量嵌入与语义匹配.md)
