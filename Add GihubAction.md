
Step 1: GitHub Repository তৈরি ও কোড আপলোড

    যদি না থাকে, GitHub এ একটা রেপো তৈরি করো
    তোমার প্রজেক্ট কোড সেখানে পুশ করো (push)

Step 2: সার্ভারে SSH Key জেনারেট করা

তোমার সার্ভার থেকে GitHub-এ কোড পুল করার জন্য SSH key লাগবে।
সার্ভারে SSH key তৈরি:

ssh root@your_server_ip
ssh-keygen -t ed25519 -C "your_email@example.com"

    এন্টার চেপে ডিফল্ট লোকেশন রাখো (~/.ssh/id_ed25519)

    পাসওয়ার্ড দিতে চাইলে দিতে পারো, অথবা এন্টার দিয়ে ফাঁকা রাখতে পারো

public key দেখো:

cat ~/.ssh/id_ed25519.pub

Step 3: GitHub রেপোতে Deploy Key হিসেবে SSH key যুক্ত করা
    GitHub এ তোমার রেপোতে যাও → Settings → Deploy keys
    Add deploy key ক্লিক করো
    Title দাও (যেমন: "My Server Key")
    Key হিসেবে পেস্ট করো তোমার public key (id_ed25519.pub এর ভিতরের content)
    Allow write access চেক করো যদি তোমার সার্ভার থেকে push করতে চাও, না হলে রাখো unchecked
    Save করো

Step 4: সার্ভারে SSH Agent চালু এবং key add করা

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

Step 5: GitHub Secrets সেট করা

GitHub Actions-এ তোমার সার্ভারে SSH করে কোড ডিপ্লয় করতে হলে তোমার সার্ভারের SSH প্রাইভেট কি GitHub এ রাখতে হবে।

    GitHub repo → Settings → Secrets and variables → Actions → New repository secret

সিক্রেটস যুক্ত করো:
Secret Name	Value
DO_SSH_KEY	সার্ভার থেকে নেওয়া private key এর content (id_ed25519 এর content)
SERVER_IP	তোমার সার্ভারের IP, যেমন: 137.184.156.194
Step 6: GitHub Actions Workflow ফাইল তৈরি করা

তোমার প্রজেক্টের রেপোতে .github/workflows/deploy.yml নামে ফাইল তৈরি করো।


name: Deploy to DigitalOcean
on:
  push:
    branches:
      - main  # যেই ব্রাঞ্চ পুশ করলে ডিপ্লয় হবে
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    - name: Set up SSH
      uses: webfactory/ssh-agent@v0.9.0
      with:
        ssh-private-key: ${{ secrets.DO_SSH_KEY }}
    - name: Deploy to Server
      run: |
        ssh -o StrictHostKeyChecking=no root@${{ secrets.SERVER_IP }} << 'EOF'
          set -e
          cd /var/www/curtcommerz
          echo "Resetting local changes..."
          git reset --hard HEAD
          git clean -fd
          echo "Pulling latest code from live branch..."
          git pull origin live
          echo "Creating virtual environment if not exists..."
          if [ ! -d "venv" ]; then
            python3 -m venv venv
          fi
          echo "Activating virtual environment..."
          source venv/bin/activate
          echo "Installing dependencies..."
          pip install -r requirements.txt
          echo "Applying migrations..."
          python manage.py migrate --noinput
          echo "Collecting static files..."
          python manage.py collectstatic --noinput
          echo "Restarting Gunicorn..."
          sudo systemctl restart gunicorn
        EOF

Step 7: Commit & Push Workflow

git add .github/workflows/deploy.yml
git commit -m "Add GitHub Action for deploy"
git push origin main
