# 抖音账户购买与自动化管理 SDK

![Python Version](https://img.shields.io/badge/python-3.8%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Last Commit](https://img.shields.io/badge/last%20commit-June%202025-brightgreen)

## 项目概述

本项目是一个用于抖音(Douyin)账户交易、自动化管理和数据分析的Python SDK。它提供了从账户购买验证到内容自动化发布的全套解决方案，适用于需要批量运营抖音账户的开发者。

**主要功能**：
- 抖音账户购买验证接口
- 账户信息自动化获取
- 内容批量发布与管理
- 数据分析与增长监控
- 安全防护与反检测机制

## 安装指南

```bash
# 克隆仓库
git clone https://github.com/yourusername/douyin-account-sdk.git
cd douyin-account-sdk

# 安装依赖
pip install -r requirements.txt

# 设置环境变量
export DOUYIN_API_KEY="your_api_key"
export DOUYIN_SECRET="your_secret"
```

## 核心功能实现

### 1. 抖音账户购买验证

```python
class DouyinAccountPurchaser:
    def __init__(self, api_key, secret):
        self.api_key = api_key
        self.secret = secret
        self.base_url = "https://api.douyin.com/v3/account/"
        
    def verify_account(self, account_id):
        """
        验证抖音账户是否可购买
        :param account_id: 抖音账户ID
        :return: 验证结果和账户信息
        """
        timestamp = int(time.time())
        params = {
            "account_id": account_id,
            "api_key": self.api_key,
            "timestamp": timestamp
        }
        params["signature"] = self._generate_signature(params)
        
        try:
            response = requests.get(
                f"{self.base_url}verify",
                params=params,
                headers={"Content-Type": "application/json"}
            )
            data = response.json()
            
            if data.get("code") == 200:
                return {
                    "status": "available",
                    "data": data["data"]
                }
            else:
                return {
                    "status": "unavailable",
                    "reason": data.get("message", "Unknown error")
                }
        except Exception as e:
            raise DouyinAPIError(f"Account verification failed: {str(e)}")
    
    def _generate_signature(self, params):
        """
        生成API请求签名
        """
        param_str = "&".join([f"{k}={v}" for k, v in sorted(params.items())])
        sign_str = param_str + self.secret
        return hashlib.sha256(sign_str.encode()).hexdigest()
```

### 2. 账户自动化管理

```python
class DouyinAccountManager:
    def __init__(self, account_data, proxy=None):
        self.account_id = account_data["account_id"]
        self.cookies = account_data.get("cookies", {})
        self.proxy = proxy
        self.session = self._init_session()
        
    def _init_session(self):
        session = requests.Session()
        if self.proxy:
            session.proxies = {"http": self.proxy, "https": self.proxy}
        
        # 设置cookies
        for name, value in self.cookies.items():
            session.cookies.set(name, value)
            
        return session
    
    def post_video(self, video_path, caption="", tags=None, schedule_time=None):
        """
        发布视频到抖音
        :param video_path: 视频文件路径
        :param caption: 视频描述
        :param tags: 标签列表
        :param schedule_time: 定时发布时间
        :return: 发布结果
        """
        upload_url = self._get_upload_url()
        video_id = self._upload_video(upload_url, video_path)
        
        payload = {
            "video_id": video_id,
            "text": caption,
            "visibility": "PUBLIC",
            "disable_comment": False,
            "disable_duet": False,
            "disable_stitch": False
        }
        
        if tags:
            payload["hashtags"] = tags
            
        if schedule_time:
            payload["schedule_time"] = int(schedule_time.timestamp())
            
        response = self.session.post(
            "https://www.douyin.com/aweme/v1/aweme/post/",
            json=payload
        )
        
        return response.json()
    
    def _get_upload_url(self):
        """获取视频上传URL"""
        response = self.session.get(
            "https://www.douyin.com/aweme/v1/upload/auth/"
        )
        return response.json()["upload_url"]
    
    def _upload_video(self, upload_url, file_path):
        """上传视频文件"""
        with open(file_path, "rb") as f:
            files = {"video": f}
            response = requests.post(upload_url, files=files)
            
        return response.json()["video_id"]
```

## 高级功能实现

### 3. 账户数据分析模块

```python
class DouyinAnalytics:
    def __init__(self, account_manager):
        self.manager = account_manager
        
    def get_account_stats(self, start_date, end_date):
        """
        获取账户统计数据
        :param start_date: 开始日期
        :param end_date: 结束日期
        :return: 统计数据
        """
        params = {
            "start_date": start_date.strftime("%Y-%m-%d"),
            "end_date": end_date.strftime("%Y-%m-%d")
        }
        
        response = self.manager.session.get(
            "https://www.douyin.com/aweme/v1/data/account/",
            params=params
        )
        
        data = response.json()
        
        return {
            "follower_count": data["follower_count"],
            "following_count": data["following_count"],
            "total_likes": data["total_likes"],
            "video_views": data["video_views"],
            "engagement_rate": data["engagement_rate"]
        }
    
    def get_video_performance(self, video_id):
        """
        获取单个视频表现数据
        """
        response = self.manager.session.get(
            f"https://www.douyin.com/aweme/v1/data/video/{video_id}/"
        )
        
        data = response.json()
        
        return {
            "views": data["play_count"],
            "likes": data["digg_count"],
            "comments": data["comment_count"],
            "shares": data["share_count"],
            "completion_rate": data["play_duration"] / data["video_duration"]
        }
    
    def generate_growth_report(self, period="weekly"):
        """
        生成账户增长报告
        """
        now = datetime.now()
        if period == "weekly":
            start_date = now - timedelta(days=7)
        elif period == "monthly":
            start_date = now - timedelta(days=30)
        else:
            start_date = now - timedelta(days=1)
            
        stats = self.get_account_stats(start_date, now)
        
        # 生成可视化报告
        plt.figure(figsize=(10, 6))
        metrics = ["follower_count", "total_likes", "video_views"]
        values = [stats[m] for m in metrics]
        
        plt.bar(metrics, values)
        plt.title(f"Douyin Account Growth Report - {period.capitalize()}")
        plt.ylabel("Count")
        plt.grid(axis="y")
        
        report_path = f"growth_report_{period}_{now.date()}.png"
        plt.savefig(report_path)
        
        return {
            "stats": stats,
            "report_path": report_path
        }
```

### 4. 安全与反检测机制

```python
class DouyinSecurity:
    def __init__(self, account_manager):
        self.manager = account_manager
        self.last_request_time = 0
        self.request_count = 0
        
    def safe_request(self, method, url, **kwargs):
        """
        安全请求封装，避免被检测为机器人
        """
        # 请求间隔控制
        current_time = time.time()
        elapsed = current_time - self.last_request_time
        
        if elapsed < 1.5:  # 最少1.5秒间隔
            sleep_time = 1.5 - elapsed
            time.sleep(sleep_time)
            
        # 请求频率限制
        if self.request_count > 30:
            time.sleep(60)  # 每分钟不超过30次请求
            self.request_count = 0
            
        # 随机请求头
        headers = kwargs.get("headers", {})
        headers.update(self._generate_random_headers())
        kwargs["headers"] = headers
        
        # 添加随机延迟
        time.sleep(random.uniform(0.1, 0.3))
        
        response = self.manager.session.request(method, url, **kwargs)
        self.last_request_time = time.time()
        self.request_count += 1
        
        # 检测是否被限制
        if response.status_code == 403 or "验证" in response.text:
            raise DouyinSecurityError("Account may be flagged, need verification")
            
        return response
    
    def _generate_random_headers(self):
        """生成随机请求头"""
        user_agents = [
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)",
            "Mozilla/5.0 (iPhone; CPU iPhone OS 15_0 like Mac OS X)",
            "Mozilla/5.0 (Linux; Android 11; SM-G975F)"
        ]
        
        return {
            "User-Agent": random.choice(user_agents),
            "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",
            "Referer": "https://www.douyin.com/",
            "X-Requested-With": "XMLHttpRequest"
        }
    
    def check_account_health(self):
        """
        检查账户健康状况
        """
        try:
            response = self.safe_request(
                "GET",
                "https://www.douyin.com/aweme/v1/user/profile/self/"
            )
            
            data = response.json()
            
            if data.get("status_code") != 0:
                return {
                    "status": "restricted",
                    "reason": data.get("status_msg", "Unknown restriction")
                }
                
            return {
                "status": "healthy",
                "follower_count": data["follower_count"],
                "aweme_count": data["aweme_count"]
            }
            
        except Exception as e:
            return {
                "status": "error",
                "reason": str(e)
            }
```

## 使用示例

### 购买并验证抖音账户

```python
# 初始化购买者
purchaser = DouyinAccountPurchaser(
    api_key="your_api_key",
    secret="your_secret"
)

# 验证账户
account_id = "1234567890"
verification = purchaser.verify_account(account_id)

if verification["status"] == "available":
    print(f"Account {account_id} is available for purchase")
    print("Account info:", verification["data"])
else:
    print(f"Account not available: {verification['reason']}")
```

### 自动化发布内容

```python
# 初始化账户管理器
account_data = {
    "account_id": "1234567890",
    "cookies": {
        "sessionid": "your_session_id",
        "ttwid": "your_ttwid"
    }
}

manager = DouyinAccountManager(account_data, proxy="http://user:pass@proxy:port")

# 发布视频
result = manager.post_video(
    video_path="/path/to/video.mp4",
    caption="Check out this awesome video! #viral #trending",
    tags=["viral", "trending", "fyp"]
)

print("Video posted:", result)
```

### 数据分析报告

```python
# 初始化分析模块
analytics = DouyinAnalytics(manager)

# 获取周报
report = analytics.generate_growth_report("weekly")
print("Weekly growth stats:", report["stats"])
print("Report saved to:", report["report_path"])

# 获取单个视频数据
video_data = analytics.get_video_performance("video_id_123")
print("Video performance:", video_data)
```

## 项目结构

```
douyin-account-sdk/
├── LICENSE
├── README.md
├── requirements.txt
├── examples/
│   ├── purchase_example.py
│   ├── posting_example.py
│   └── analytics_example.py
├── douyin_sdk/
│   ├── __init__.py
│   ├── purchaser.py       # 账户购买模块
│   ├── manager.py         # 账户管理模块
│   ├── analytics.py       # 数据分析模块
│   ├── security.py        # 安全模块
│   └── exceptions.py      # 自定义异常
└── tests/
    ├── test_purchaser.py
    ├── test_manager.py
    └── test_analytics.py
```

## 贡献指南

我们欢迎贡献！请遵循以下步骤：

1. Fork 本项目
2. 创建您的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交您的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 打开 Pull Request

## 许可证

本项目采用 MIT 许可证 - 详情请参阅 [LICENSE](LICENSE) 文件。

## 免责声明

本项目仅供学习和研究使用，请遵守抖音的用户协议和相关法律法规。开发者不对任何滥用行为负责。

---

**最后更新**: June 2025  
**维护者**: [Your Name]  
**联系方式**: your.email@example.com
