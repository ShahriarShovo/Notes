✅ Step-by-step Django Deployment with Gunicorn + Nginx on DigitalOcean Droplet (Ubuntu)

🛠️ Prerequisites:
    1.Ubuntu Droplet (tested on 20.04/22.04)
    2.SSH access
    3.Your Django project (let’s say project name: dfcms)
    4.You’ve already created a Droplet and SSH into it.


✅ 1. Update the System & Install Packages
        sudo apt update && sudo apt upgrade -y
        sudo apt install python3-pip python3-dev python3-venv nginx curl git -y


✅ 2. Clone or Upload Your Django Project
        cd /var/www/
        sudo git clone https://github.com/yourusername/yourproject.git
        sudo chown -R $USER:$USER yourproject
        cd yourproject
        Note: Alternatively: Use scp, rsync, or sftp to upload your code if you didn’t use git.

✅ 3. Set Up Virtual Environment
        python3 -m venv venv
        source venv/bin/activate
        pip install --upgrade pip
        pip install -r requirements.txt

✅ 4. Set Up Django Settings
        DEBUG = False
        ALLOWED_HOSTS = ['your_server_ip', 'yourdomain.com']
        STATIC_ROOT = BASE_DIR / 'static/'

✅ 5. Collect Static Files
        python manage.py collectstatic --noinput

✅ 6. Run Migrations & Create Superuser (optional)
        python manage.py migrate
        python manage.py createsuperuser

✅ 7. Test with Gunicorn
        gunicorn --bind 127.0.0.1:8000 core.wsgi:application
        If that works, press Ctrl+C.
        if gunicorn run already:
        ✅ Option 1: Stop Gunicorn process
            check : sudo lsof -i :8000
            🛑 Kill: sudo fuser -k 8000/tcp
            🛑 start : gunicorn --bind 127.0.0.1:8000 core.wsgi:application

        ✅ Option 2: User another port:

            gunicorn --bind 127.0.0.1:8010 core.wsgi:application
            Note : Need to set proxy_pass or Nginx config same prot 8010

        ✅ Option 3: If Gunicorn service already started :
            1. sudo systemctl status gunicorn
            2. if started then restart :
                sudo systemctl restart gunicorn or  kill the old instance : sudo pkill gunicorn

        🔁 Bonus: If you use .socket :
            sudo systemctl stop gunicorn.socket
            sudo systemctl stop gunicorn.service
            sudo systemctl start gunicorn.service

        🧪 Verify : curl 127.0.0.1:8000


✅ 8. Set up Gunicorn as a systemd Service
        Create Gunicorn service file:
        sudo nano /etc/systemd/system/gunicorn.service
        [Unit]
        Description=gunicorn daemon for Django project
        After=network.target
        [Service]
        User=yourusername
        Group=www-data
        WorkingDirectory=/var/www/dfcms
        ExecStart=/var/www/dfcms/venv/bin/gunicorn --access-logfile - --workers 3       --bind 127.0.0.1:8000 core.wsgi:application
        [Install]
        WantedBy=multi-user.target

        Then run:
            sudo systemctl daemon-reexec
            sudo systemctl daemon-reload
            sudo systemctl start gunicorn
            sudo systemctl enable gunicorn
            Check status:
            sudo systemctl status gunicorn


✅ 9. Configure Nginx
        sudo nano /etc/nginx/sites-available/yourproject
        Paste:
            server {
                listen 80;
                server_name yourdomain.com your_server_ip;
                location = /favicon.ico { access_log off; log_not_found off; }
                location /static/ {
                root /var/www/yourproject;  // give the correct path for static files//
                }
                location / {
                    include proxy_params;
                    proxy_pass http://127.0.0.1:8000;
                }
            }

        Enable the site:
        sudo ln -s /etc/nginx/sites-available/dfcms /etc/nginx/sites-enabled/
        sudo nginx -t
        sudo systemctl restart nginx


✅ 10. Allow Firewall
        sudo ufw allow 'Nginx Full'

✅ 11. Optional – Set up Domain & SSL (Let's Encrypt)
        sudo apt install certbot python3-certbot-nginx -y
        sudo certbot --nginx -d yourdomain.com


✅ Deployment Done 🎉
    Now visit:
        http://yourdomain.com or
        http://your_server_ip

Your Django project should be live!







