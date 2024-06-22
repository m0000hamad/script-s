#!/bin/bash

# Check if the script is run as root
if [[ $EUID -ne 0 ]]; then
   echo "This script must be run as root"
   exit 1
fi

# Function to validate cron schedule format
validate_cron_schedule() {
    local cron_schedule="$1"
    if [[ ! "$cron_schedule" =~ ^([0-9]|[1-5][0-9]|\*|(\*\/[1-9][0-9]*))\ +([0-9]|1[0-9]|2[0-3]|\*|(\*\/[1-9][0-9]*))\ +([1-9]|[12][0-9]|3[01]|\*|(\*\/[1-9][0-9]*))\ +([1-9]|1[0-2]|\*|(\*\/[1-9][0-9]*))\ +([0-6]|\*|(\*\/[1-9][0-9]*))$ ]]; then
        echo "Invalid cron schedule format"
        return 1
    fi
    return 0
}

# Function to create RSA key pair
create_ssh_key() {
    local ssh_key_path="$1"
    ssh-keygen -t rsa -b 4096 -N "" -f "$ssh_key_path"
}

# Get domain name, internal server IP, Cloudflare email, and API token from user input
read -p "Enter the domain name: " DOMAIN
read -p "Enter the internal server IP: " INTERNAL_IP
read -p "Enter the SSH port for the internal server: " SSH_PORT
read -p "Enter your Cloudflare email: " CF_EMAIL
read -p "Enter your Cloudflare API token: " CF_API_TOKEN

# Ask user for root password of internal server
read -sp "Enter the root password for the internal server: " ROOT_PASS
echo

# Ask user for directory to store certificates on the internal server
read -p "Enter the directory to store certificates on the internal server (default: /root): " DEST_DIR
DEST_DIR=${DEST_DIR:-/root}

# Get file names for storing certificates
read -p "Enter the certificate file name (default: cert.crt): " CERT_FILE
CERT_FILE=${CERT_FILE:-cert.crt}
read -p "Enter the private key file name (default: private.key): " KEY_FILE
KEY_FILE=${KEY_FILE:-private.key}

# Get cron job schedule for renewal or use default (monthly)
read -p "Enter the cron job schedule for renewal (default: '0 0 1 * *' for monthly): " CRON_SCHEDULE
CRON_SCHEDULE=${CRON_SCHEDULE:-'0 0 1 * *'}

# Validate cron schedule format
validate_cron_schedule "$CRON_SCHEDULE" || exit 1

# Create SSH key pair if not already exists
SSH_KEY_PATH="/root/.ssh/id_rsa"
if [ ! -f "$SSH_KEY_PATH" ]; then
    echo "Creating SSH key pair..."
    create_ssh_key "$SSH_KEY_PATH"
else
    echo "SSH key pair already exists."
fi

# Add SSH key to the authorized_keys of the internal server
echo "Adding SSH key to the authorized_keys of the internal server..."
sshpass -p "$ROOT_PASS" ssh-copy-id -i "$SSH_KEY_PATH.pub" -p $SSH_PORT root@$INTERNAL_IP

# Update the system and install necessary packages
echo "Updating the system and installing Certbot..."
apt-get update
apt-get install -y certbot python3-certbot-dns-cloudflare openssh-client sshpass

# Create Cloudflare configuration file
CF_INI_PATH="/root/.secrets/certbot/cloudflare.ini"
mkdir -p /root/.secrets/certbot
echo "dns_cloudflare_email = $CF_EMAIL" > $CF_INI_PATH
echo "dns_cloudflare_api_key = $CF_API_TOKEN" >> $CF_INI_PATH
chmod 600 $CF_INI_PATH

# Obtain the certificate for the domain using Cloudflare DNS challenge
echo "Obtaining the certificate for $DOMAIN using Cloudflare DNS challenge..."
if ! certbot certonly --dns-cloudflare --dns-cloudflare-credentials $CF_INI_PATH -d $DOMAIN; then
    echo "Error obtaining certificate. Please check your Cloudflare credentials."
    exit 1
fi

# Define certificate directory
CERT_DIR="/etc/letsencrypt/live/$DOMAIN"

# Add the internal server to known_hosts
echo "Adding the internal server to known_hosts..."
ssh-keyscan -H -p $SSH_PORT $INTERNAL_IP >> ~/.ssh/known_hosts

# Transfer the certificates to the internal server
echo "Transferring certificates to the internal server $INTERNAL_IP..."
if ! sshpass -p "$ROOT_PASS" scp -P $SSH_PORT -i "$SSH_KEY_PATH" $CERT_DIR/fullchain.pem root@$INTERNAL_IP:$DEST_DIR/$CERT_FILE; then
    echo "Error transferring certificate file. Please check your SSH setup."
    exit 1
fi
if ! sshpass -p "$ROOT_PASS" scp -P $SSH_PORT -i "$SSH_KEY_PATH" $CERT_DIR/privkey.pem root@$INTERNAL_IP:$DEST_DIR/$KEY_FILE; then
    echo "Error transferring private key file. Please check your SSH setup."
    exit 1
fi

# Create a cron job for automatic renewal and transfer
echo "Creating a cron job for automatic renewal..."
(crontab -l 2>/dev/null; echo "$CRON_SCHEDULE certbot renew --quiet --deploy-hook 'sshpass -p \"$ROOT_PASS\" scp -P $SSH_PORT -i \"$SSH_KEY_PATH\" $CERT_DIR/fullchain.pem root@$INTERNAL_IP:$DEST_DIR/$CERT_FILE && sshpass -p \"$ROOT_PASS\" scp -P $SSH_PORT -i \"$SSH_KEY_PATH\" $CERT_DIR/privkey.pem root@$INTERNAL_IP:$DEST_DIR/$KEY_FILE'") | crontab -

echo "Certificate setup and configuration completed."
echo "This script was created by m0000hamad."
