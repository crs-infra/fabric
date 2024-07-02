# fabric
use fabric to transcribe and summarise youtube contents
import os
import subprocess
import requests
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Retrieve YouTube API key, channel IDs, and email configuration from environment variables
YOUTUBE_API_KEY = os.getenv('YOUTUBE_API_KEY')
EMAIL_ADDRESS = os.getenv('EMAIL_ADDRESS')
EMAIL_PASSWORD = os.getenv('EMAIL_PASSWORD')
TO_EMAIL_ADDRESS = os.getenv('TO_EMAIL_ADDRESS')

CHANNEL_IDS = {
    'HC': os.getenv('HOC_Channel_ID'),
    'NS': os.getenv('NOBS_Channel_ID'),
    'CC': os.getenv('CC_Channel_ID'),
    'JP': os.getenv('JP_Channel_ID'),
    'LD': os.getenv('LD_Channel_ID'),
    'VB': os.getenv('VB_Channel_ID'),
    'DB': os.getenv('DB_Channel_ID')
}

def get_latest_video_url(api_key, channel_id):
    """Get the latest video URL from a YouTube channel."""
    url = f"https://www.googleapis.com/youtube/v3/search?key={api_key}&channelId={channel_id}&order=date&part=snippet&type=video&maxResults=1"
    response = requests.get(url)
    data = response.json()
    video_id = data['items'][0]['id']['videoId']
    return f"https://www.youtube.com/watch?v={video_id}"

def process_channel(name, channel_id):
    """Process a YouTube channel to extract wisdom from the latest video and save the file."""
    latest_video_url = get_latest_video_url(YOUTUBE_API_KEY, channel_id)
    print(latest_video_url)
    command = f"yt --transcript {latest_video_url} | fabric --stream --pattern extract_wisdom | save -s {name}"
    os.system(command)
    return f"{name}.md"  # the save command saves the file as {name}.txt

def send_email(subject, body, files):
    """Send an email with the specified subject, body, and attached files."""
    msg = MIMEMultipart()
    msg['From'] = EMAIL_ADDRESS
    msg['To'] = TO_EMAIL_ADDRESS
    msg['Subject'] = subject

    msg.attach(MIMEText(body, 'plain'))

    for file in files:
        attachment = open(file, "rb")

        part = MIMEBase('application', 'octet-stream')
        part.set_payload((attachment).read())
        encoders.encode_base64(part)
        part.add_header('Content-Disposition', f"attachment; filename= {file}")

        msg.attach(part)

    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login(EMAIL_ADDRESS, EMAIL_PASSWORD)
    text = msg.as_string()
    server.sendmail(EMAIL_ADDRESS, TO_EMAIL_ADDRESS, text)
    server.quit()

# Process each channel and collect the generated file names
generated_files = []
for name, channel_id in CHANNEL_IDS.items():
    file_name = process_channel(name, channel_id)
    generated_files.append(file_name)

# Send the email with the generated files attached
send_email("Extracted Wisdom Files", "Please find the attached files with extracted wisdom.", generated_files)
