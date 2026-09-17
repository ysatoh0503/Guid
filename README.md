# Guidfrom 
flask import Flask, render_template, request, send_file
import imaplib
import email
import re
import csv
import io
import os

app = Flask(__name__)

IMAP_SERVER = "imap.gmail.com"
EMAIL_USER = os.getenv("MAIL_USER")
EMAIL_PASS = os.getenv("MAIL_PASS")
KEYWORD = "GUID"

def extract_records():
    mail = imaplib.IMAP4_SSL(IMAP_SERVER)
    mail.login(EMAIL_USER, EMAIL_PASS)
    mail.select("INBOX")

    result, data = mail.search(None, f'(BODY "{KEYWORD}")')
    mail_ids = data[0].split()

    records = []

    for num in mail_ids:
        result, data = mail.fetch(num, "(RFC822)")
        raw_email = data[0][1]
        msg = email.message_from_bytes(raw_email)

        if msg.is_multipart():
            for part in msg.walk():
                if part.get_content_type() == "text/plain":
                    body = part.get_payload(decode=True).decode("utf-8", errors="ignore")
        else:
            body = msg.get_payload(decode=True).decode("utf-8", errors="ignore")

        name_match = re.search(r"氏名[:：]\s*(.+)", body)
        guid_match = re.search(r"GUID[:：]\s*([0-9A-Za-z\-]+)", body)

        name = name_match.group(1).strip() if name_match else ""
        guid = guid_match.group(1).strip() if guid_match else ""

        if name or guid:
            records.append([name, guid])

    return records

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/extract")
def extract():
    records = extract_records()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["氏名", "GUID"])
    writer.writerows(records)

    output.seek(0)
    return send_file(
        io.BytesIO(output.getvalue().encode("utf-8")),
        mimetype="text/csv",
        as_attachment=True,
        download_name="mail_list.csv"
    )

if __name__ == "__main__":
    app.run()

    <!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>メール抽出アプリ</title>
</head>
<body>
    <h2>氏名・GUID 抽出アプリ</h2>
    <p>メールから自動で一覧を作成します。</p>
    <a href="/extract">
        <button style="font-size:20px;">抽出してCSVを作成</button>
    </a>
</body>
</html>
