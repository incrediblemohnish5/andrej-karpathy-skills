# 示例

展示四条原则的真实世界代码示例。每个示例都会说明 LLM 常见的错误做法，以及应该如何修正。

---

## 1. 编码前先思考

### 示例 1：隐藏的假设

**用户请求：** “添加一个导出用户数据的功能”

**❌ LLM 常见做法（错误假设）**

```python
def export_users(format='json'):
    """Export all users to a file."""
    users = User.query.all()
    
    if format == 'json':
        with open('users.json', 'w') as f:
            json.dump([u.to_dict() for u in users], f)
    elif format == 'csv':
        with open('users.csv', 'w') as f:
            writer = csv.DictWriter(f, fieldnames=['id', 'email', 'name'])
            writer.writeheader()
            writer.writerows([u.to_dict() for u in users])
    
    return f"Exported {len(users)} users"
```

**问题：**
- 默认应该导出“所有”用户，但没有确认（分页怎么办？隐私怎么办？）
- 没问就假定了文件保存位置
- 没问就假定了导出哪些字段
- 没检查真实数据结构，就假定了 CSV 的字段名

**✅ 正确做法（显式暴露假设）**

```
在实现之前，我需要先澄清几点：

1. **范围**：导出所有用户，还是导出筛选后的子集？（涉及隐私）
2. **形式**：你说的“导出”具体是指：
   - 浏览器里下载文件？
   - 后台任务，完成后邮件通知？
   - 返回数据的 API endpoint？
3. **字段**：要导出哪些用户字段？（有些字段可能敏感）
4. **规模**：通常有多少用户？（这会影响实现方式）

最简单的方案是：添加一个返回分页 JSON 的 API endpoint。
如果要做基于文件的导出，还需要更多信息。

你倾向哪种方式？
```

### 示例 2：多种可能解释

**用户请求：** “让搜索更快”

**❌ LLM 常见做法（默默选一种）**

```python
# 不先确认“更快”具体指什么，
# 直接加缓存、数据库索引和异步处理
@lru_cache(maxsize=1000)
async def search(query: str) -> List[Result]:
    # ... 200 lines of optimization code
```

**✅ 正确做法**

```
“让搜索更快”可能有几种不同含义：

1. **响应时间更快**（例如从 500ms 降到 100ms 以内）
   - 增加数据库索引
   - 缓存高频查询
   - 预估工作量：2 小时

2. **能承受更多并发搜索**（吞吐量）
   - 增加 async/await
   - 连接池
   - 预估工作量：4 小时

3. **用户感知上更快**（体验层面）
   - 先显示部分结果
   - 渐进式加载
   - 预估工作量：3 小时

当前典型查询耗时约为 500ms。
对你的场景来说，哪一项最重要？
```

---

## 2. 简单优先

### 示例 1：过度抽象

**用户请求：** “加一个计算折扣的函数”

**❌ LLM 常见做法（过度设计）**

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Protocol, Union
from dataclasses import dataclass

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float:
        pass

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percentage: float):
        self.percentage = percentage
    
    def calculate(self, amount: float) -> float:
        return amount * (self.percentage / 100)

class FixedDiscount(DiscountStrategy):
    def __init__(self, fixed_amount: float):
        self.fixed_amount = fixed_amount
    
    def calculate(self, amount: float) -> float:
        return min(self.fixed_amount, amount)

@dataclass
class DiscountConfig:
    strategy: DiscountStrategy
    min_purchase: float = 0.0
    max_discount: float = float('inf')

class DiscountCalculator:
    def __init__(self, config: DiscountConfig):
        self.config = config
    
    def apply_discount(self, amount: float) -> float:
        if amount < self.config.min_purchase:
            return 0
        discount = self.config.strategy.calculate(amount)
        return min(discount, self.config.max_discount)

# 为了一个简单计算，使用时却要先写 30+ 行配置代码
```

**✅ 正确做法（简单）**

```python
def calculate_discount(amount: float, percent: float) -> float:
    """Calculate discount amount. percent should be 0-100."""
    return amount * (percent / 100)

# Usage
discount = calculate_discount(100.0, 10.0)  # $10 off
```

**什么时候再加复杂性：** 只有当你真的需要支持多种折扣类型时。如果这个需求以后出现，再重构。

### 示例 2：预设性功能

**用户请求：** “把用户偏好保存到数据库”

**❌ LLM 常见做法（加入无关功能）**

```python
class PreferenceManager:
    def __init__(self, db, cache=None, validator=None):
        self.db = db
        self.cache = cache or InMemoryCache()
        self.validator = validator or DefaultValidator()
    
    def save(self, user_id: int, prefs: dict, 
             merge: bool = True, 
             validate: bool = True,
             notify: bool = False) -> bool:
        """Save preferences with optional merging, validation, and notifications."""
        
        if validate:
            errors = self.validator.validate(prefs)
            if errors:
                raise ValidationError(errors)
        
        existing = self.get(user_id) if merge else {}
        
        if merge:
            existing.update(prefs)
            prefs = existing
        
        self.db.execute(
            "UPDATE users SET preferences = ? WHERE id = ?",
            (json.dumps(prefs), user_id)
        )
        
        if self.cache:
            self.cache.set(f"prefs:{user_id}", prefs)
        
        if notify:
            self.notify_preference_change(user_id, prefs)
        
        return True
    
    def notify_preference_change(self, user_id: int, prefs: dict):
        # Another 30 lines for a feature nobody asked for
        pass
```

**✅ 正确做法（只做被要求的事）**

```python
def save_preferences(db, user_id: int, preferences: dict):
    """Save user preferences to database."""
    db.execute(
        "UPDATE users SET preferences = ? WHERE id = ?",
        (json.dumps(preferences), user_id)
    )
```

**以后再加即可：** 缓存（当性能真成问题时）、校验（当坏数据真的出现时）、合并逻辑（当需求明确提出时）。

---

## 3. 外科手术式修改

### 示例 1：顺手重构

**用户请求：** “修复空邮箱会让校验器崩溃的问题”

**❌ LLM 常见做法（改太多）**

```diff
  def validate_user(user_data):
-     # Check email format
-     if not user_data.get('email'):
+     """Validate user data."""
+     email = user_data.get('email', '').strip()
+     
+     # Validate email
+     if not email:
          raise ValueError("Email required")
-     
-     # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email or '.' not in email.split('@')[1]:
          raise ValueError("Invalid email")
      
-     # Check username
-     if not user_data.get('username'):
+     # Validate username
+     username = user_data.get('username', '').strip()
+     if not username:
          raise ValueError("Username required")
+     if len(username) < 3:
+         raise ValueError("Username too short")
+     if not username.isalnum():
+         raise ValueError("Username must be alphanumeric")
      
      return True
```

**问题：**
- 除了修 bug，还“顺手加强”了邮箱校验
- 加了用户根本没要求的用户名校验
- 改了注释
- 加了 docstring

**✅ 正确做法（外科手术式修改）**

```diff
  def validate_user(user_data):
      # Check email format
-     if not user_data.get('email'):
+     email = user_data.get('email', '')
+     if not email or not email.strip():
          raise ValueError("Email required")
      
      # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email:
          raise ValueError("Invalid email")
      
      # Check username
      if not user_data.get('username'):
          raise ValueError("Username required")
      
      return True
```

**唯一改动：** 只改了修复空邮箱处理所必需的那几行。

### 示例 2：风格漂移

**用户请求：** “给上传函数加日志”

**❌ LLM 常见做法（顺带重排一切）**

```diff
- def upload_file(file_path, destination):
+ def upload_file(file_path: str, destination: str) -> bool:
+     """Upload file to destination with logging."""
+     logger.info(f"Uploading {file_path} to {destination}")
+     
      try:
-         with open(file_path, 'rb') as f:
-             data = f.read()
-         
-         response = requests.post(destination, files={'file': data})
-         
-         if response.status_code == 200:
-             return True
-         else:
-             return False
+         with open(file_path, "rb") as f:
+             data = f.read()
+         response = requests.post(destination, files={"file": data})
+         success = response.status_code == 200
+         
+         if success:
+             logger.info(f"Upload successful: {file_path}")
+         else:
+             logger.error(f"Upload failed: {file_path}, status={response.status_code}")
+         
+         return success
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f"Upload error for {file_path}: {e}")
          return False
```

**问题：**
- 改了引号风格（`''` 变成 `""`）
- 加了用户没要求的类型标注
- 加了 docstring
- 重排了空白格式
- 顺便改了布尔返回逻辑

**✅ 正确做法（保持现有风格）**

```diff
+ import logging
+ 
+ logger = logging.getLogger(__name__)
+ 
  def upload_file(file_path, destination):
+     logger.info(f'Starting upload: {file_path}')
      try:
          with open(file_path, 'rb') as f:
              data = f.read()
          
          response = requests.post(destination, files={'file': data})
          
          if response.status_code == 200:
+             logger.info(f'Upload successful: {file_path}')
              return True
          else:
+             logger.error(f'Upload failed: {file_path}, status={response.status_code}')
              return False
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f'Upload error: {file_path}')
          return False
```

**保持一致的点：** 单引号、不加类型标注、维持原来的布尔分支写法、保持原有间距风格。

---

## 4. 目标驱动执行

### 示例 1：模糊 vs 可验证

**用户请求：** “修复认证系统”

**❌ LLM 常见做法（太模糊）**

```
我会这样修复认证系统：
1. 查看代码
2. 找出问题
3. 做一些改进
4. 测试修改

[在没有清晰成功标准的情况下直接开始改]
```

**✅ 正确做法（可验证目标）**

```
要“修复认证”，我需要先定义成功标准。
我们要解决的具体问题是什么？

如果问题是“用户修改密码后仍然保持登录”：

计划：
1. 写测试：修改密码 → 验证旧 session 被失效
   验证：测试先失败（成功复现 bug）

2. 实现：修改密码时使 session 失效
   验证：测试通过

3. 检查边界情况：多个活跃 session、并发修改
   验证：附加测试通过

4. 验证无回归：现有认证测试仍全部通过
   验证：完整测试套件为绿色

当前认证相关测试覆盖率：[展示覆盖率]
你现在遇到的具体认证问题是什么？
```

### 示例 2：多步骤并带验证

**用户请求：** “给 API 加限流”

**❌ LLM 常见做法（一次性全上）**

```python
# 一次提交里实现完整限流系统：
# Redis、多种策略、配置系统、监控，总共 300 行
# 没有清晰的验证步骤
```

**✅ 正确做法（增量推进并验证）**

```
限流方案计划：

1. 先加基础的内存限流（单个 endpoint）
   验证：
   - 测试：100 次请求 → 前 10 次成功，其余返回 429
   - 手动：curl 同一个 endpoint 11 次，看到限流错误

2. 抽成中间件（应用到所有 endpoints）
   验证：
   - 测试：/users 和 /posts 都会被限流
   - 现有 endpoint 测试仍通过

3. 加 Redis 后端（支持多实例）
   验证：
   - 测试：应用重启后限流状态仍正确
   - 测试：两个应用实例共享同一个限流计数器

4. 加配置能力（不同 endpoint 不同速率）
   验证：
   - 测试：/search 允许 10 次/分钟，/users 允许 100 次/分钟
   - 配置文件可被正确解析

每一步都可以独立验证，也可以独立上线。
先从第 1 步开始？
```

### 示例 3：先测试再修复

**用户请求：** “有重复分数时，排序会出问题”

**❌ LLM 常见做法（不先复现就修）**

```python
# 没确认 bug 是否存在，就直接修改排序逻辑
def sort_scores(scores):
    return sorted(scores, key=lambda x: (-x['score'], x['name']))
```

**✅ 正确做法（先复现）**

```python
# 1. 先写一个能复现问题的测试
def test_sort_with_duplicate_scores():
    """Test sorting when multiple items have same score."""
    scores = [
        {'name': 'Alice', 'score': 100},
        {'name': 'Bob', 'score': 100},
        {'name': 'Charlie', 'score': 90},
    ]
    
    result = sort_scores(scores)
    
    # 这个 bug：重复分数时顺序不稳定
    # 多跑几次测试，结果应该保持一致
    assert result[0]['score'] == 100
    assert result[1]['score'] == 100
    assert result[2]['score'] == 90

# 验证：把测试跑 10 次 → 会因顺序不一致而失败

# 2. 现在再用稳定排序修复
def sort_scores(scores):
    """Sort by score descending, then name ascending for ties."""
    return sorted(scores, key=lambda x: (-x['score'], x['name']))

# 验证：测试稳定通过
```

---

## 反模式总结

| 原则 | 反模式 | 修正方式 |
|-----------|-------------|-----|
| 编码前先思考 | 默默假定文件格式、字段、范围 | 显式列出假设，并请求澄清 |
| 简单优先 | 为单一折扣计算引入策略模式 | 在确实需要复杂性前，只保留一个函数 |
| 外科手术式修改 | 修 bug 时顺带改引号风格、加类型标注 | 只改能修复已报告问题的那些行 |
| 目标驱动执行 | “我先看看代码然后改进一下” | “先为 bug X 写测试 → 让测试通过 → 验证无回归” |

## 关键洞察

那些“过度复杂”的示例并不是显而易见地错误，它们甚至用了设计模式和最佳实践。问题在于 **时机**：它们在真正需要之前就提前引入复杂性，这会：

- 让代码更难理解
- 引入更多 bug
- 增加实现时间
- 让测试变得更难

而“简单”的版本则：
- 更容易理解
- 更快实现
- 更容易测试
- 当复杂性真的出现时，之后再重构也完全来得及

**好代码，是用尽可能简单的方式解决今天的问题，而不是过早解决明天的问题。**
