### SMTP Configuration:
After going to Google account settings (https://myaccount.google.com/apppasswords), generate an 'app password'. Then in grafana's compose yaml:

    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.gmail.com:587
      - GF_SMTP_USER=9mahin.ahmad@gmail.com
      - GF_SMTP_PASSWORD=16-char-app-password from https://myaccount.google.com/apppasswords
      - GF_SMTP_FROM_ADDRESS=your-email@gmail.com
      - GF_SMTP_FROM_NAME=Grafana Alerting
      - GF_SMTP_STARTTLS_POLICY=Mandatory

### Create a "Contact Point" and an "Alert rule"

