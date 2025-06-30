# jimeng-ai-
火山引擎api使用即梦ai图生视频

https://chat.deepseek.com/a/chat/s/de6c3a67-1a76-4084-9285-21eabf958576

文档：

https://www.volcengine.com/docs/85621/1544774 

文档首页/即梦AI/接入说明/即梦AI-视频生成/即梦AI-图生视频S2.0Pro


火山引擎 即梦
https://console.volcengine.com/ai/ability/detail/10 开通试用api↑

（开通图片api可以图生视频，视频api只能文生视频）

https://console.volcengine.com/ai/ability/info/103

计费面板↑

https://console.volcengine.com/iam/keymanage

开通key和secret↑

jimeng_config.py ：

```python
#jimeng_config.py

ACCESS_KEY = "AK"  # 替换为实际Access Key ID
SECRET_KEY = "WX=="  # 替换为实际 Secret Access Key
#上面两行参数在 https://console.volcengine.com/iam/keymanage 开启api密钥获取↑

DEFAULT_IMAGE_URL = "https://ogre.natalie.mu/media/news/comic/2025/0628/mugenjo_poster_ver2.jpg"  # 替换为实际图片URL

DEFAULT_PROMPT = "人物缓缓望着镜头，希区柯克式变焦，人物与背景移速不同"

LOG_FILE = "jimeng_video.log"  # 日志文件路径

DEBUG_MODE = True

MAX_RETRIES = 3

```

```python
import json
import time
import requests
import datetime
import hashlib
import hmac
import logging
import struct
from jimeng_config import *

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

class ImageHeaderParser:
    """通过图片二进制头解析分辨率（无PIL依赖）"""
    
    @staticmethod
    def get_dimensions(url):
        """获取图片宽高并返回最接近的标准比例"""
        try:
            # 只下载图片头部（JPEG/PNG的尺寸信息通常在前100字节）
            headers = {'Range': 'bytes=0-999'}
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            data = response.content

            # 检测JPEG (以FF D8开头)
            if data.startswith(b'\xff\xd8'):
                # JPEG的SOF0标记(FFC0)后第5-8字节是宽高
                sof_offset = data.find(b'\xff\xc0')
                if sof_offset != -1 and len(data) >= sof_offset + 9:
                    height, width = struct.unpack('>HH', data[sof_offset+5:sof_offset+9])
                    return width, height

            # 检测PNG (以89 50 4E 47开头)
            elif data.startswith(b'\x89PNG'):
                # PNG的第16-24字节是宽高
                if len(data) >= 24:
                    width = int.from_bytes(data[16:20], 'big')
                    height = int.from_bytes(data[20:24], 'big')
                    return width, height

            logger.warning("⚠️ 仅支持JPEG/PNG格式，使用默认比例")
            return None
            
        except Exception as e:
            logger.warning(f"⚠️ 分辨率检测失败: {str(e)}")
            return None

    @staticmethod
    def get_recommended_ratio(url):
        """获取推荐比例"""
        dimensions = ImageHeaderParser.get_dimensions(url)
        if not dimensions:
            return "16:9"
            
        width, height = dimensions
        STANDARD_RATIOS = {
            '16:9': 1.7778,
            '4:3': 1.3333,
            '1:1': 1.0,
            '3:4': 0.75,
            '9:16': 0.5625,
            '21:9': 2.3333,
            '9:21': 0.4286
        }
        
        current_ratio = width / height
        closest = min(STANDARD_RATIOS.items(), 
                     key=lambda x: abs(x[1] - current_ratio))
        
        logger.info(f"📐 图片分辨率: {width}x{height} → 推荐比例: {closest[0]}")
        return closest[0]

class SignatureV4:
    """签名工具类"""
    
    @staticmethod
    def get_signature_key(key, date_stamp, region, service):
        k_date = hmac.new(key.encode(), date_stamp.encode(), hashlib.sha256).digest()
        k_region = hmac.new(k_date, region.encode(), hashlib.sha256).digest()
        k_service = hmac.new(k_region, service.encode(), hashlib.sha256).digest()
        return hmac.new(k_service, b'request', hashlib.sha256).digest()

class JimengVideoAPI:
    def __init__(self):
        self.host = "visual.volcengineapi.com"
        self.region = "cn-north-1"
        self.service = "cv"
    
    def _make_request(self, action, body, retry_count=0):
        """执行带签名的API请求"""
        try:
            # 1. 准备时间戳
            t = datetime.datetime.utcnow()
            amz_date = t.strftime('%Y%m%dT%H%M%SZ')
            date_stamp = t.strftime('%Y%m%d')
            
            # 2. 生成签名
            credential_scope = f"{date_stamp}/{self.region}/{self.service}/request"
            payload_hash = hashlib.sha256(json.dumps(body).encode()).hexdigest()
            
            # 规范请求
            canonical_headers = '\n'.join([
                f"content-type:application/json",
                f"host:{self.host}",
                f"x-content-sha256:{payload_hash}",
                f"x-date:{amz_date}"
            ])
            
            canonical_request = '\n'.join([
                'POST',
                '/',
                f"Action={action}&Version=2022-08-31",
                canonical_headers,
                '',
                'content-type;host;x-content-sha256;x-date',
                payload_hash
            ])
            
            # 计算签名
            string_to_sign = '\n'.join([
                'HMAC-SHA256',
                amz_date,
                credential_scope,
                hashlib.sha256(canonical_request.encode()).hexdigest()
            ])
            
            signing_key = SignatureV4.get_signature_key(
                SECRET_KEY, date_stamp, self.region, self.service
            )
            signature = hmac.new(signing_key, string_to_sign.encode(), hashlib.sha256).hexdigest()
            
            # 3. 构建请求头
            headers = {
                'X-Date': amz_date,
                'Authorization': (
                    f"HMAC-SHA256 Credential={ACCESS_KEY}/{credential_scope}, "
                    f"SignedHeaders=content-type;host;x-content-sha256;x-date, "
                    f"Signature={signature}"
                ),
                'X-Content-Sha256': payload_hash,
                'Content-Type': 'application/json'
            }
            
            # 4. 发送请求
            response = requests.post(
                f"https://{self.host}?Action={action}&Version=2022-08-31",
                headers=headers,
                json=body,
                timeout=30
            )
            
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.HTTPError as e:
            if retry_count < MAX_RETRIES and e.response.status_code >= 500:
                wait_time = 2 ** retry_count
                logger.warning(f"🔄 请求失败，{wait_time}秒后重试... (剩余 {MAX_RETRIES-retry_count}次)")
                time.sleep(wait_time)
                return self._make_request(action, body, retry_count + 1)
                
            error_msg = f"HTTP错误 {e.response.status_code}: "
            try:
                error_msg += json.dumps(e.response.json(), indent=2, ensure_ascii=False)
            except:
                error_msg += e.response.text[:200]
            raise Exception(error_msg)
            
        except Exception as e:
            if retry_count < MAX_RETRIES:
                return self._make_request(action, body, retry_count + 1)
            raise

class JimengVideoGenerator:
    def __init__(self, image_url=None, prompt=None):
        self.api = JimengVideoAPI()
        self.image_url = image_url or DEFAULT_IMAGE_URL
        self.prompt = prompt or DEFAULT_PROMPT
        
    def generate(self):
        """生成视频完整流程"""
        try:
            # 1. 检测图片比例
            aspect_ratio = ImageHeaderParser.get_recommended_ratio(self.image_url)
            
            # 2. 提交任务
            logger.info("🚀 提交视频生成任务...")
            submit_res = self.api._make_request(
                "CVSync2AsyncSubmitTask",
                {
                    "req_key": "jimeng_vgfm_i2v_l20",
                    "image_urls": [self.image_url],
                    "prompt": self.prompt,
                    "aspect_ratio": aspect_ratio,
                    "seed": -1
                }
            )
            
            task_id = submit_res["data"]["task_id"]
            logger.info(f"✅ 任务已提交 | 任务ID: {task_id}")
            
            # 3. 轮询结果（带进度动画）
            logger.info("\n⏳ 正在生成视频，请稍候...")
            for i in range(1, 31):  # 最多轮询30次
                result = self.api._make_request(
                    "CVSync2AsyncGetResult",
                    {"req_key": "jimeng_vgfm_i2v_l20", "task_id": task_id}
                )
                
                status = result["data"].get("status")
                progress = result["data"].get("progress", 0)
                
                # 动态进度条
                progress_bar = f"[{'=' * int(progress//3)}{' ' * (33-int(progress//3))}]"
                logger.info(f"  进度: {progress_bar} {progress}% (轮询 {i}/30)")
                
                if status == "done":
                    video_url = result["data"].get("video_url")
                    if video_url:
                        logger.info(f"\n🎉 生成成功！视频链接（1小时内有效）:\n{video_url}")
                        return video_url
                
                elif status == "failed":
                    error = result["data"].get("error", "未知错误")
                    raise Exception(f"生成失败: {error}")
                
                time.sleep(5)
                
            raise TimeoutError("⏱️ 生成超时，请稍后手动查询任务ID")
            
        except Exception as e:
            logger.error(f"\n❌ 生成失败: {str(e)}")
            return None

if __name__ == "__main__":
    try:
        generator = JimengVideoGenerator(
            # image_url="您的自定义图片URL",
            # prompt="您的自定义提示词"
        )
        
        if video_url := generator.generate():
            logger.info("✨ 请及时下载视频")
        else:
            logger.error("💥 生成失败，请检查日志: " + LOG_FILE)
            
    except Exception as e:
        logger.exception("程序异常终止")
```
