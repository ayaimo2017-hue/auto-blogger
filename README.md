تحديث الكود وتشغيل Auto Blogger Arabic PRO
name: Auto Blogger Arabic PRO

on:
  schedule:
    - cron: "*/5 * * * *"  # تشغيل كل 5 دقائق
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: إعداد البيئة
        run: |
          sudo apt update
          sudo apt install python3 python3-pip imagemagick -y
          pip install requests beautifulsoup4 Pillow

      - name: إنشاء مجلد docs
        run: mkdir -p docs

      - name: إنشاء مقالات تلقائية احترافية
        run: |
          python3 - <<'PY'
          import requests, random, os, re
          from bs4 import BeautifulSoup
          from datetime import datetime
          from PIL import Image, ImageDraw, ImageFont
          from io import BytesIO

          sources = [
              "https://www.youm7.com/",
              "https://mobizil.com/",
              "https://www.masrawy.com/"
          ]

          keywords = ["موبيزل", "سعر", "مراجعة", "هاتف", "موبايل", "مواصفات", "مقارنة"]

          def clean_text(text):
              text = re.sub(r'\s+', ' ', text)
              return text.strip()

          def fetch_article(url):
              try:
                  html = requests.get(url, timeout=10).text
                  soup = BeautifulSoup(html, "html.parser")
                  title = soup.title.text if soup.title else "مقال جديد"
                  paras = ' '.join([p.text for p in soup.find_all('p')[:15]])
                  imgs = [img['src'] for img in soup.find_all('img', src=True) if img['src'].startswith('http')]
                  if len(paras) < 400: return None, None, None
                  content = clean_text(paras)
                  return title, content, imgs[:1]
              except:
                  return None, None, None

          def rewrite_title(title):
              title = re.sub(r'مواصفات|سعر|مراجعة', 'تفاصيل جديدة لهاتف موبيزل', title)
              title += f" | مقارنة ومواصفات من Mobizl Play"
              return title

          def add_watermark(image_url, text="Mobizl Play"):
              try:
                  resp = requests.get(image_url, timeout=10)
                  img = Image.open(BytesIO(resp.content)).convert("RGBA")
                  watermark = Image.new("RGBA", img.size, (255,255,255,0))
                  draw = ImageDraw.Draw(watermark)
                  font = ImageFont.load_default()
                  draw.text((10, img.size[1]-30), text, (255,0,0,180), font=font)
                  combined = Image.alpha_composite(img, watermark)
                  output_path = f"docs/wm_{random.randint(1000,9999)}.png"
                  combined.convert("RGB").save(output_path)
                  return output_path
              except:
                  return None

          os.makedirs("docs", exist_ok=True)

          all_articles = []
          for site in sources:
              try:
                  html = requests.get(site, timeout=10).text
                  soup = BeautifulSoup(html, "html.parser")
                  links = [a['href'] for a in soup.find_all('a', href=True) if 'http' in a['href']]
                  random.shuffle(links)
                  for link in links[:2]:
                      title, content, imgs = fetch_article(link)
                      if not title or not content: continue
                      title = rewrite_title(title)
                      img_path = add_watermark(imgs[0]) if imgs else None
                      file_name = f"docs/article_{random.randint(1000,9999)}.md"
                      with open(file_name, "w", encoding="utf-8") as f:
                          f.write(f"# {title}\n\n{content}\n\n")
                          if img_path: f.write(f"![صورة]({img_path})\n")
                      print("✅ تم إنشاء مقال:", file_name)
              except Exception as e:
                  print("خطأ:", e)

          PY

      - name: رفع المقالات الجديدة إلى GitHub
        run: |
          git config --global user.name "auto-bot"
          git config --global user.email "actions@github.com"
          git add docs/
          git commit -m "📰 إضافة مقالات جديدة تلقائيًا" || echo "لا توجد تغييرات"
          git push
