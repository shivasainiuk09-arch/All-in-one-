# main.py
from flask import Flask, request, jsonify, send_from_directory
from datetime import datetime, timedelta
import os, openai

app = Flask(__name__)
openai.api_key = os.environ.get("OPENAI_API_KEY")  # Replit secret

# Settings
TRIAL_DAYS = 7
FREE_ADS_PER_DAY = 1
PAID_PLAN_PRICE = 3000
PAID_PLAN_DURATION_DAYS = 60
PAID_PLAN_TOTAL_ADS = 5000
PAID_PLAN_ADS_PER_MONTH = 2500

# Users DB
users = {}  # email: {plan, trial_start, ads_generated_today, last_ad_date, paid_start, paid_ads_remaining}

def can_generate_ad(user):
    today = datetime.now().date()
    if user['plan'] == 'paid':
        if (today - user['paid_start'].date()).days >= PAID_PLAN_DURATION_DAYS:
            return False, "Paid plan expired. Renew to continue generating ads.", True
        if user['paid_ads_remaining'] <= 0:
            return False, "You reached 2,500 ads this month. Wait for next month or renew plan.", True
        return True, "", False
    trial_end = user['trial_start'].date() + timedelta(days=TRIAL_DAYS)
    if today > trial_end:
        return False, f"Trial expired. Upgrade to paid plan ₹{PAID_PLAN_PRICE} for 2 months / 5,000 ads total.", True
    if user.get('last_ad_date') != today:
        user['ads_generated_today'] = 0
        user['last_ad_date'] = today
    if user['ads_generated_today'] >= FREE_ADS_PER_DAY:
        return False, "Daily free ad limit reached. Try tomorrow.", False
    return True, "", False

@app.route('/')
def index():
    return send_from_directory('.', 'index.html')

@app.route('/generate', methods=['POST'])
def generate():
    data = request.get_json()
    email = data.get('email')
    businessName = data.get('businessName')
    businessType = data.get('businessType')
    promoText = data.get('promoText')
    phone = data.get('phone')

    if email not in users:
        users[email] = {"plan":"trial","trial_start":datetime.now(),"ads_generated_today":0,"last_ad_date":datetime.now().date()}
    user = users[email]

    ok, msg, showUpgrade = can_generate_ad(user)
    if not ok:
        return jsonify({"error": msg, "showUpgrade": showUpgrade})

    # AI prompt
    prompt = f"Create a short catchy WhatsApp/Instagram ad for {businessName} ({businessType}) with promo: {promoText}. Contact: {phone}, Email: {email}."
    try:
        response = openai.Completion.create(
            model="text-davinci-003",
            prompt=prompt,
            max_tokens=120
        )
        ad_text = response.choices[0].text.strip()
        if user['plan']=='trial':
            user['ads_generated_today'] += 1
        else:
            user['paid_ads_remaining'] -=1
        return jsonify({"ad": ad_text})
    except Exception as e:
        return jsonify({"error":"Error generating ad. Try again.", "showUpgrade": False})

@app.route('/activate_paid', methods=['POST'])
def activate_paid():
    data = request.get_json()
    email = data.get('email')
    if email not in users:
        return jsonify({"success":False,"message":"User not found"})
    user = users[email]
    user['plan'] = 'paid'
    user['paid_start'] = datetime.now()
    user['paid_ads_remaining'] = PAID_PLAN_ADS_PER_MONTH
    return jsonify({"success":True,"message":f"Paid plan activated! ₹{PAID_PLAN_PRICE}, 2 months / 5,000 ads total (2,500/month)."})

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=3000)<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AI WhatsApp Ad Generator</title>
<style>
body {font-family:Arial; padding:20px; max-width:500px; margin:auto; background:#f0f0f0;}
input, textarea, button {width:100%; margin:10px 0; padding:10px; font-size:16px;}
button {background:#007bff; color:white; border:none; cursor:pointer;}
button:hover {background:#0056b3;}
#adOutput {margin-top:20px; background:white; padding:15px; border-radius:8px; display:none;}
</style>
</head>
<body>

<h2>AI WhatsApp Ad Generator</h2>

<div id="userForm">
    <input type="email" id="email" placeholder="Your Email">
    <input type="text" id="phone" placeholder="Phone Number">
    <input type="text" id="businessName" placeholder="Business Name">
    <input type="text" id="businessType" placeholder="Business Type">
    <textarea id="promoText" placeholder="Promo / Offer"></textarea>
    <button onclick="submitForm()">Generate Ad</button>
</div>

<div id="adOutput">
    <h3>Generated Ad:</h3>
    <p id="adText"></p>
    <a id="downloadLink" download="ad.txt">Download Ad</a><br><br>
    <a id="whatsappLink" target="_blank">Share on WhatsApp</a>
</div>

<script>
let currentEmail = '';
async function submitForm(){
    const email = document.getElementById('email').value;
    const phone = document.getElementById('phone').value;
    const businessName = document.getElementById('businessName').value;
    const businessType = document.getElementById('businessType').value;
    const promoText = document.getElementById('promoText').value;

    if(!email || !phone || !businessName || !businessType || !promoText){
        alert('Fill all fields'); return;
    }
    currentEmail = email;

    const res = await fetch('/generate',{
        method:'POST',
        headers:{'Content-Type':'application/json'},
        body:JSON.stringify({email, phone, businessName, businessType, promoText})
    });
    const data = await res.json();

    if(data.error){
        alert(data.error);
        if(data.showUpgrade){
            if(confirm("Upgrade to paid plan ₹3,000 for 2 months / 5,000 ads?")){
                await fetch('/activate_paid',{
                    method:'POST',
                    headers:{'Content-Type':'application/json'},
                    body:JSON.stringify({email})
                });
                alert('Paid plan activated! Generate ads now.');
            }
        }
        return;
    }

    document.getElementById('adText').innerText = data.ad;
    document.getElementById('adOutput').style.display='block';
    const blob = new Blob([data.ad],{type:'text/plain'});
    document.getElementById('downloadLink').href = URL.createObjectURL(blob);

    // WhatsApp share
    const waText = encodeURIComponent(data.ad);
    document.getElementById('whatsappLink').href = `https://wa.me/?text=${waText}`;
    document.getElementById('whatsappLink').innerText = "Share on WhatsApp";
}
</script>

</body>
</html>
