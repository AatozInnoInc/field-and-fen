# External Service Integrations Specification

## Overview

This document specifies how to integrate with external APIs and services used in the Field and Fen automation pipeline.

---

## Table of Contents

1. [Replicate (Image Upscaling)](#replicate)
2. [Google Drive (File Storage)](#google-drive)
3. [Shopify (E-commerce)](#shopify)
4. [Printful (Fulfillment)](#printful)
5. [Etsy (Marketplace)](#etsy)
6. [Meta Business Suite (Instagram/Facebook)](#meta)
7. [TikTok (Video Posts)](#tiktok)
8. [Pinterest (Pins)](#pinterest)

---

## Replicate

**Purpose:** AI-powered image upscaling (72dpi → 300dpi, print-ready)

### API Details

- **Base URL:** `https://api.replicate.com/v1`
- **Auth:** Bearer token in `Authorization` header
- **Docs:** https://replicate.com/docs/reference/http

### Model Selection

Use **Real-ESRGAN** model for photo upscaling:
- Model ID: `nightmareai/real-esrgan:42fed1c4974146d4d2414e2be2c5277c7fcf05fcc3a73abf41610695738c1d7b`
- Supports 2x, 4x, 8x upscaling
- Good for photographs (vs illustrations)

### Client Implementation

**File:** `app/services/replicate.py`

```python
import replicate
from app.config import settings
import requests
from typing import Optional

class ReplicateClient:
    def __init__(self):
        self.client = replicate.Client(api_token=settings.replicate_api_token)
    
    def upscale_image(
        self,
        input_path: str,
        scale: int = 4,
        face_enhance: bool = False
    ) -> str:
        """
        Upscale an image using Real-ESRGAN.
        
        Args:
            input_path: Local path to input image
            scale: Upscale factor (2, 4, or 8)
            face_enhance: Enable face enhancement (for portraits)
        
        Returns:
            URL of upscaled image (temporary, download immediately)
        
        Raises:
            ReplicateError: On API failure
        """
        with open(input_path, "rb") as f:
            output = self.client.run(
                "nightmareai/real-esrgan:42fed1c4974146d4d2414e2be2c5277c7fcf05fcc3a73abf41610695738c1d7b",
                input={
                    "image": f,
                    "scale": scale,
                    "face_enhance": face_enhance
                }
            )
        
        # Output is a URL (valid for ~1 hour)
        if isinstance(output, list):
            return output[0]
        return output
    
    def download_result(self, url: str, output_path: str):
        """Download upscaled image from Replicate CDN."""
        response = requests.get(url, stream=True)
        response.raise_for_status()
        
        with open(output_path, "wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
```

### Usage Example

```python
from app.services.replicate import ReplicateClient

client = ReplicateClient()

# Upscale image
output_url = client.upscale_image(
    input_path="/tmp/original.jpg",
    scale=4
)

# Download result
client.download_result(output_url, "/tmp/upscaled.jpg")
```

### Cost

- **Pricing:** ~$0.01-0.05 per image (varies by model/scale)
- **Free Tier:** $10 credit on signup
- **Track usage:** Log to `api_usage` table

### Error Handling

```python
try:
    output_url = client.upscale_image(input_path)
except replicate.exceptions.ReplicateError as e:
    logger.error(f"Replicate API error: {str(e)}")
    raise
```

---

## Google Drive

**Purpose:** File storage for source images and processed outputs

### API Details

- **Auth:** OAuth 2.0 with Service Account
- **Scopes:** `https://www.googleapis.com/auth/drive`
- **SDK:** google-api-python-client

### Setup

1. Create Service Account in Google Cloud Console
2. Download JSON credentials
3. Share Google Drive folder with service account email
4. Store credentials at path in `GOOGLE_DRIVE_CREDENTIALS_PATH`

### Client Implementation

**File:** `app/services/google_drive.py`

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload, MediaIoBaseDownload
from app.config import settings
import io

class GoogleDriveClient:
    def __init__(self):
        credentials = service_account.Credentials.from_service_account_file(
            settings.google_drive_credentials_path,
            scopes=['https://www.googleapis.com/auth/drive']
        )
        self.service = build('drive', 'v3', credentials=credentials)
    
    def list_files(self, folder_id: str, page_size: int = 100):
        """List files in a folder."""
        query = f"'{folder_id}' in parents and trashed=false"
        results = self.service.files().list(
            q=query,
            pageSize=page_size,
            fields="files(id, name, mimeType, createdTime)"
        ).execute()
        return results.get('files', [])
    
    def download_file(self, file_id: str, output_path: str):
        """Download a file by ID."""
        request = self.service.files().get_media(fileId=file_id)
        
        with io.FileIO(output_path, 'wb') as fh:
            downloader = MediaIoBaseDownload(fh, request)
            done = False
            while not done:
                status, done = downloader.next_chunk()
    
    def upload_file(
        self,
        file_path: str,
        filename: str,
        folder_id: str,
        mime_type: str = 'image/jpeg'
    ) -> str:
        """
        Upload a file to Google Drive.
        
        Returns:
            file_id of uploaded file
        """
        file_metadata = {
            'name': filename,
            'parents': [folder_id]
        }
        media = MediaFileUpload(file_path, mimetype=mime_type)
        
        file = self.service.files().create(
            body=file_metadata,
            media_body=media,
            fields='id'
        ).execute()
        
        return file.get('id')
    
    def move_file(self, file_id: str, new_parent_id: str):
        """Move a file to a different folder."""
        # Get current parents
        file = self.service.files().get(
            fileId=file_id,
            fields='parents'
        ).execute()
        previous_parents = ",".join(file.get('parents', []))
        
        # Move file
        self.service.files().update(
            fileId=file_id,
            addParents=new_parent_id,
            removeParents=previous_parents,
            fields='id, parents'
        ).execute()
```

### Folder Structure

Store folder IDs in config or database:

```python
DRIVE_FOLDERS = {
    "incoming": "1abc123...",
    "ready_to_process": "1def456...",
    "published": "1ghi789...",
    "species": {
        "whitetail-deer": "1jkl012...",
        "fox": "1mno345...",
        # ... other species
    }
}
```

---

## Shopify

**Purpose:** Primary e-commerce platform

### API Details

- **Base URL:** `https://{store}.myshopify.com/admin/api/2024-01`
- **Auth:** Custom App with Admin API access token
- **Docs:** https://shopify.dev/docs/api/admin-rest

### Setup

1. Create Custom App in Shopify Admin
2. Grant scopes: `read_products`, `write_products`, `read_orders`
3. Copy Admin API access token

### Client Implementation

**File:** `app/services/shopify.py`

```python
import requests
from app.config import settings
from typing import Dict, Any, List

class ShopifyClient:
    def __init__(self):
        self.base_url = f"https://{settings.shopify_store_url}/admin/api/2024-01"
        self.headers = {
            "X-Shopify-Access-Token": settings.shopify_api_key,
            "Content-Type": "application/json"
        }
    
    def create_product(
        self,
        title: str,
        description: str,
        images: List[str],
        variants: List[Dict[str, Any]],
        tags: List[str] = None,
        product_type: str = "Canvas Print"
    ) -> Dict[str, Any]:
        """
        Create a product with variants.
        
        Args:
            title: Product title
            description: HTML description
            images: List of image URLs
            variants: List of variant dicts with price, sku, etc
            tags: Product tags
            product_type: Product type/category
        
        Returns:
            Created product data
        """
        payload = {
            "product": {
                "title": title,
                "body_html": description,
                "product_type": product_type,
                "tags": ",".join(tags) if tags else "",
                "images": [{"src": url} for url in images],
                "variants": variants
            }
        }
        
        response = requests.post(
            f"{self.base_url}/products.json",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        
        return response.json()["product"]
    
    def update_product(self, product_id: str, updates: Dict[str, Any]):
        """Update product fields."""
        payload = {"product": updates}
        
        response = requests.put(
            f"{self.base_url}/products/{product_id}.json",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        
        return response.json()["product"]
    
    def get_product(self, product_id: str) -> Dict[str, Any]:
        """Fetch product by ID."""
        response = requests.get(
            f"{self.base_url}/products/{product_id}.json",
            headers=self.headers
        )
        response.raise_for_status()
        
        return response.json()["product"]
    
    def upload_image(self, product_id: str, image_url: str):
        """Add image to existing product."""
        payload = {
            "image": {
                "src": image_url
            }
        }
        
        response = requests.post(
            f"{self.base_url}/products/{product_id}/images.json",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        
        return response.json()["image"]
```

### Variant Structure

For canvas prints with multiple sizes:

```python
variants = [
    {
        "option1": "12×8",
        "price": "29.99",
        "sku": "WHITETAIL-001-12x8",
        "inventory_management": "shopify",
        "inventory_quantity": 999
    },
    {
        "option1": "24×16",
        "price": "59.99",
        "sku": "WHITETAIL-001-24x16",
        "inventory_management": "shopify",
        "inventory_quantity": 999
    },
    # ... more sizes
]
```

### Rate Limits

- **REST Admin API:** 2 requests/second (burst: 40)
- **Handle 429 errors:** Retry after `Retry-After` header seconds

```python
import time

def handle_rate_limit(response):
    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", 2))
        time.sleep(retry_after)
        return True
    return False
```

---

## Printful

**Purpose:** Print-on-demand fulfillment (integrated with Shopify)

### API Details

- **Base URL:** `https://api.printful.com`
- **Auth:** Bearer token in `Authorization` header
- **Docs:** https://developers.printful.com/docs/

### Sync Strategy

Printful syncs with Shopify automatically once connected. No direct API integration needed for most use cases.

**When to use Printful API:**
1. Check order status
2. Get shipping estimates
3. Access product catalog (to set correct variant SKUs)

### Client Implementation (Optional)

**File:** `app/services/printful.py`

```python
import requests
from app.config import settings

class PrintfulClient:
    def __init__(self):
        self.base_url = "https://api.printful.com"
        self.headers = {
            "Authorization": f"Bearer {settings.printful_api_token}",
            "Content-Type": "application/json"
        }
    
    def get_products(self):
        """List available Printful products."""
        response = requests.get(
            f"{self.base_url}/products",
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()["result"]
    
    def get_variant_info(self, product_id: int):
        """Get product variants (sizes, prices)."""
        response = requests.get(
            f"{self.base_url}/products/{product_id}",
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()["result"]
```

**Product IDs (Canvas Prints):**
- Canvas 12×8: Product ID 48
- Canvas 24×16: Product ID 48
- (Check Printful docs for full list)

---

## Etsy

**Purpose:** Marketplace for handmade/art products

### API Details

- **Base URL:** `https://openapi.etsy.com/v3`
- **Auth:** OAuth 2.0 (PKCE flow)
- **Docs:** https://developers.etsy.com/documentation/

### OAuth Flow

1. Direct user to authorization URL
2. User approves, redirected with auth code
3. Exchange code for access token
4. Store refresh token (expires in 90 days)

**Implementation:** Use `requests-oauthlib` or `authlib`

### Client Implementation

**File:** `app/services/etsy.py`

```python
import requests
from app.config import settings

class EtsyClient:
    def __init__(self, access_token: str):
        self.base_url = "https://openapi.etsy.com/v3"
        self.headers = {
            "Authorization": f"Bearer {access_token}",
            "x-api-key": settings.etsy_api_key
        }
    
    def create_listing(
        self,
        shop_id: str,
        title: str,
        description: str,
        price: float,
        quantity: int,
        images: List[str],
        tags: List[str]
    ) -> Dict[str, Any]:
        """
        Create a draft listing.
        
        Note: Max 13 tags, each 20 chars max.
        """
        payload = {
            "quantity": quantity,
            "title": title[:140],  # Max 140 chars
            "description": description,
            "price": price,
            "who_made": "i_did",
            "when_made": "2020_2024",
            "taxonomy_id": 1111,  # Art > Prints (check taxonomy API)
            "tags": tags[:13]  # Max 13 tags
        }
        
        response = requests.post(
            f"{self.base_url}/application/shops/{shop_id}/listings",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        
        listing = response.json()
        
        # Upload images (separate API calls)
        for image_url in images:
            self.upload_listing_image(shop_id, listing["listing_id"], image_url)
        
        return listing
    
    def upload_listing_image(self, shop_id: str, listing_id: str, image_url: str):
        """Upload image to listing."""
        # Download image first (Etsy requires multipart form upload)
        image_data = requests.get(image_url).content
        
        files = {
            'image': ('image.jpg', image_data, 'image/jpeg')
        }
        
        response = requests.post(
            f"{self.base_url}/application/shops/{shop_id}/listings/{listing_id}/images",
            files=files,
            headers={"Authorization": self.headers["Authorization"]}
        )
        response.raise_for_status()
        return response.json()
```

### Rate Limits

- **10 requests/second**
- Use rate limiting decorator or sleep between calls

---

## Meta Business Suite

**Purpose:** Post to Instagram and Facebook

### API Details

- **Base URL:** `https://graph.facebook.com/v18.0`
- **Auth:** Long-lived access token
- **Docs:** https://developers.facebook.com/docs/instagram-api

### Setup

1. Create Facebook App
2. Add Instagram Graph API product
3. Connect Instagram Business Account
4. Generate long-lived token (60 days, refresh programmatically)

### Client Implementation

**File:** `app/services/meta.py`

```python
import requests
from app.config import settings
from typing import List

class MetaClient:
    def __init__(self):
        self.base_url = "https://graph.facebook.com/v18.0"
        self.access_token = settings.meta_access_token
        self.instagram_account_id = settings.instagram_account_id
    
    def post_image(
        self,
        image_url: str,
        caption: str,
        hashtags: List[str] = None
    ) -> str:
        """
        Post image to Instagram feed.
        
        Returns:
            Post ID
        """
        if hashtags:
            caption += "\n\n" + " ".join(f"#{tag}" for tag in hashtags)
        
        # Step 1: Create media container
        response = requests.post(
            f"{self.base_url}/{self.instagram_account_id}/media",
            params={
                "image_url": image_url,
                "caption": caption,
                "access_token": self.access_token
            }
        )
        response.raise_for_status()
        container_id = response.json()["id"]
        
        # Step 2: Publish media
        response = requests.post(
            f"{self.base_url}/{self.instagram_account_id}/media_publish",
            params={
                "creation_id": container_id,
                "access_token": self.access_token
            }
        )
        response.raise_for_status()
        
        return response.json()["id"]
    
    def post_story(self, image_url: str) -> str:
        """Post to Instagram Stories."""
        response = requests.post(
            f"{self.base_url}/{self.instagram_account_id}/media",
            params={
                "image_url": image_url,
                "media_type": "STORIES",
                "access_token": self.access_token
            }
        )
        response.raise_for_status()
        container_id = response.json()["id"]
        
        response = requests.post(
            f"{self.base_url}/{self.instagram_account_id}/media_publish",
            params={
                "creation_id": container_id,
                "access_token": self.access_token
            }
        )
        response.raise_for_status()
        
        return response.json()["id"]
```

### Rate Limits

- **25 posts per day** (Instagram feed)
- **No limit on stories** (but avoid spam)

---

## TikTok

**Purpose:** Short-form video posts

### API Details

- **Base URL:** `https://open.tiktokapis.com/v2`
- **Auth:** OAuth 2.0
- **Docs:** https://developers.tiktok.com/doc/content-posting-api-get-started

### Video Requirements

- Format: MP4
- Duration: 3-60 seconds
- Resolution: 720p minimum
- Size: <4GB

### Client Implementation (Placeholder)

TikTok API access requires approval. Once approved:

```python
class TikTokClient:
    def post_video(self, video_url: str, caption: str, hashtags: List[str]):
        # Implementation after API approval
        pass
```

---

## Pinterest

**Purpose:** Visual discovery, drive traffic to shop

### API Details

- **Base URL:** `https://api.pinterest.com/v5`
- **Auth:** OAuth 2.0
- **Docs:** https://developers.pinterest.com/docs/api/v5/

### Client Implementation

**File:** `app/services/pinterest.py`

```python
import requests
from app.config import settings

class PinterestClient:
    def __init__(self, access_token: str):
        self.base_url = "https://api.pinterest.com/v5"
        self.headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        }
    
    def create_pin(
        self,
        board_id: str,
        title: str,
        description: str,
        image_url: str,
        link: str
    ) -> Dict[str, Any]:
        """Create a pin with link to product."""
        payload = {
            "board_id": board_id,
            "title": title,
            "description": description,
            "media_source": {
                "source_type": "image_url",
                "url": image_url
            },
            "link": link  # URL to Shopify product
        }
        
        response = requests.post(
            f"{self.base_url}/pins",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        
        return response.json()
```

---

## Summary: Service Client Pattern

All service clients follow this pattern:

1. **Configuration via environment variables** (API keys, tokens)
2. **Error handling with retries** (exponential backoff)
3. **Logging to `api_usage` table** (track calls, rate limits)
4. **Type hints and docstrings** (for maintainability)

**Decorator for API logging:**

```python
from functools import wraps
from app.database import SessionLocal
from app.models.api_usage import ApiUsage

def log_api_call(service: str, endpoint: str = None):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            
            # Log to database
            db = SessionLocal()
            db.add(ApiUsage(service=service, endpoint=endpoint or func.__name__))
            db.commit()
            db.close()
            
            return result
        return wrapper
    return decorator

# Usage:
@log_api_call(service="shopify", endpoint="create_product")
def create_product(...):
    ...
```

---

## Next Steps

After implementing these integrations:
1. Test each client with mock data
2. Add retry logic and error handling
3. Implement rate limiting (avoid hitting API limits)
4. Log all API calls to `api_usage` table
5. Monitor costs (especially Replicate, charged per use)
