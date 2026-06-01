# Deployment Guide

## Production Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] Environment variables configured
- [ ] Database backed up
- [ ] SSL certificates ready
- [ ] Domain registered
- [ ] Dependencies updated

## Backend Deployment

### Option 1: Deploy to Render (Used for this project)

#### Step 1: Create Render Account
- Go to https://render.com
- Sign up with GitHub

#### Step 2: Create Backend Web Service
1. In Render, click **New +** > **Web Service**.
2. Connect the GitHub repository.
3. Set the root directory to `backend`.
4. Use these settings:
   - Runtime: Python
   - Build Command: `pip install -r requirements.txt`
   - Start Command: `gunicorn "app:create_app()"`

#### Step 3: Configure Backend Environment Variables
Add these variables in Render > Environment:
```
FLASK_ENV=production
FLASK_DEBUG=False
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your_supabase_anon_key
SUPABASE_SERVICE_KEY=your_supabase_service_role_key
JWT_SECRET_KEY=your_secure_secret_key
UPLOAD_FOLDER=./uploads
ALLOWED_EXTENSIONS=pdf,jpg,jpeg,png,doc,docx
FLASK_PORT=10000
```

#### Step 4: Deploy
- Click **Create Web Service**.
- Render builds and deploys automatically from GitHub.
- Check **Logs** if the service fails to start.

**Backend URL:** `https://your-render-backend.onrender.com`

### Option 2: Deploy to AWS

#### Step 1: Create EC2 Instance
- Instance type: t3.micro (free tier)
- OS: Ubuntu 20.04

#### Step 2: Connect via SSH
```bash
ssh -i your-key.pem ubuntu@your-instance-ip
```

#### Step 3: Install Dependencies
```bash
sudo apt update
sudo apt install python3-pip python3-venv nginx

# Clone repo
git clone your-repo-url
cd admission/backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

#### Step 4: Configure Nginx
Create `/etc/nginx/sites-available/default`:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### Step 5: Run with Gunicorn
```bash
gunicorn -w 4 -b 0.0.0.0:5000 app:create_app()
```

### Option 3: Deploy with Docker

Create `Dockerfile` in backend:
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:create_app()"]
```

Build and run:
```bash
docker build -t admission-backend .
docker run -p 5000:5000 -e SUPABASE_URL=... admission-backend
```

## Frontend Deployment

### Option 1: Deploy to Render Static Site (Used for this project)

#### Step 1: Create Static Site
1. In Render, click **New +** > **Static Site**.
2. Connect the GitHub repository.
3. Set the root directory to `frontend`.
4. Use these settings:
   - Build Command: `npm install && npm run build`
   - Publish Directory: `dist`

#### Step 2: Configure Frontend Environment Variables
Add this variable in Render > Environment:
```
VITE_API_BASE_URL=https://your-render-backend.onrender.com/api
```

#### Step 3: Deploy
- Click **Create Static Site**.
- Render builds and deploys automatically from GitHub.
- Visit the Render static site URL after deployment.

### Option 2: Deploy to Netlify

#### Step 1: Build Frontend
```bash
cd frontend
npm run build
```

#### Step 2: Deploy
1. Go to https://netlify.com
2. Drag and drop `dist` folder
3. Add environment variable for API URL
4. Deploy

### Option 3: Deploy to GitHub Pages

```bash
cd frontend
npm run build

# Deploy dist folder to gh-pages branch
npm install --save-dev gh-pages
```

### Option 4: AWS S3 + CloudFront

#### Step 1: Create S3 Bucket
```bash
aws s3 mb s3://your-app-name --region us-east-1
```

#### Step 2: Build and Upload
```bash
cd frontend
npm run build

aws s3 sync dist/ s3://your-app-name --delete
```

#### Step 3: Set up CloudFront
- Create CloudFront distribution
- Point to S3 bucket
- Use your domain

## Database Setup (Supabase)

### Production Database

1. Create new Supabase project for production
2. Get production credentials
3. Run database schema:
   ```sql
   -- Paste database_schema.sql content
   ```

### Database Backup
```sql
-- Backup before deploying
pg_dump -h db.supabase.co -U postgres -d postgres > backup.sql
```

## Environment Variables

### Backend Production (.env)
```
FLASK_ENV=production
FLASK_DEBUG=False
SUPABASE_URL=https://your-prod.supabase.co
SUPABASE_KEY=your_prod_key
SUPABASE_SERVICE_KEY=your_prod_service_key
JWT_SECRET_KEY=your-very-secure-secret-key
UPLOAD_FOLDER=/var/www/admission/uploads
ALLOWED_EXTENSIONS=pdf,jpg,jpeg,png,doc,docx
FLASK_PORT=5000
```

### Frontend Production (.env)
```
VITE_API_BASE_URL=https://your-backend-domain.com/api
VITE_APP_NAME=Admission Management System
```

## SSL/HTTPS Setup

### Free SSL with Let's Encrypt
```bash
sudo apt install certbot python3-certbot-nginx

sudo certbot certonly --nginx -d your-domain.com

# Auto-renew
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

## Performance Optimization

### Frontend Optimization
```bash
# Minify and optimize
npm run build

# Use Gzip compression
# In Render Static Sites: Auto-enabled
# In Netlify: Auto-enabled
# In nginx: 
# gzip on;
# gzip_types text/plain text/javascript;
```

### Backend Optimization
```python
# Use connection pooling
# Use caching (Redis)
# Enable compression
```

### Database Optimization
```sql
-- Create indexes
CREATE INDEX idx_students_email ON students(email);
CREATE INDEX idx_students_status ON students(status);
```

## Monitoring & Logging

### Monitor Backend
```bash
# Render
# Use the Logs tab in the Render dashboard.

# AWS/Linux
tail -f /var/log/app.log
```

### Monitor Frontend
- Use Render metrics/logs
- Use browser DevTools

### Setup Error Tracking
- Add Sentry integration
- Setup CloudWatch alerts

## Scaling Strategy

### Phase 1: Initial Launch
- Single backend server
- CDN for frontend

### Phase 2: Growth
- Load balancer
- Multiple backend instances
- Database replication

### Phase 3: Enterprise
- Multi-region deployment
- Advanced caching
- Auto-scaling groups

## Domain Configuration

### Update DNS Records
```
A Record: your-domain.com → your-server-ip
CNAME Record: www.your-domain.com → your-domain.com
```

### SSL Certificate
- Auto with Render Static Sites
- Manual with Let's Encrypt on AWS

## Backup & Recovery

### Daily Backup
```bash
# Database backup
pg_dump ... > backup_$(date +%Y%m%d).sql

# Store in S3
aws s3 cp backup.sql s3://your-backups/
```

### Recovery
```bash
psql -h host -U user < backup.sql
```

## Security Checklist

- [ ] HTTPS enabled
- [ ] Environment variables secured
- [ ] JWT secret strong
- [ ] Password hashing verified
- [ ] CORS properly configured
- [ ] SQL injection prevented
- [ ] XSS protection enabled
- [ ] Rate limiting implemented
- [ ] Firewall configured
- [ ] Regular backups scheduled

## Post-Deployment

### Testing
1. Test all features in production
2. Verify API endpoints
3. Check database connectivity
4. Monitor performance

### Monitoring
- Set up error alerts
- Monitor server resources
- Track API performance
- Review user analytics

### Updates
- Plan maintenance windows
- Test updates in staging
- Deploy during low traffic
- Have rollback plan

## Troubleshooting

### Backend Won't Start
```
- Check environment variables
- Check Supabase connection
- Review error logs
- Check port availability
```

### Frontend Won't Connect
```
- Verify API URL in .env
- Check CORS settings
- Verify network connectivity
- Check browser console
```

### Database Connection Failed
```
- Verify Supabase URL
- Check credentials
- Verify firewall rules
- Test connection string
```

## Cost Estimation

| Service | Tier | Cost |
|---------|------|------|
| Supabase | Starter | Free |
| Render | Free/Starter | Free or paid plan |
| AWS EC2 | t3.micro | Free (1 year) |
| Domain | .com | ~$10/year |

**Total (Basic):** ~$25/month

## Support & Maintenance

### Regular Tasks
- Monitor logs (daily)
- Check backups (daily)
- Update dependencies (weekly)
- Review security (monthly)
- Capacity planning (quarterly)

### Escalation Process
1. Monitor alerts
2. Check logs
3. Restart service
4. Rollback if needed
5. Contact support

---

**Last Updated:** April 22, 2026
**Version:** 1.0.0

For questions, refer to:
- Provider documentation
- README.md
- PROJECT_SUMMARY.md
