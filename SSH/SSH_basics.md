# SSH Basics

## What is SSH?

SSH (Secure Shell) lets you remotely connect to another computer and control it via the command line. This is how you connect to GCP VMs, remote servers, or even another laptop on the same network.

---

## Enable SSH on Your Mac

Before you can SSH into your Mac, you need to enable Remote Login.

- Enable Remote Login :
  - Go to System Settings > General > Sharing
  - Toggle Remote Login ON
  - Set "Allow access for" : All users or specific users
- Test it works locally : `ssh localhost`

---

## SSH Into Another Computer on the Same Network

- Find your Mac's IP address : `ipconfig getifaddr en0`
- Find your Mac username (if unknown) : `whoami`
- Connect from another computer : `ssh username@ip_address`
- Navigate to your directory : `cd /path/to/your/folder`

---

## SSH Into a GCP VM

- Using gcloud : `gcloud compute ssh USERNAME@VM_NAME --zone ZONE`
- Using standard SSH with Google key : `ssh -i ~/.ssh/google_compute_engine USERNAME@EXTERNAL_IP`

---

## SSH Keys — No Password Every Time

SSH keys let you connect without typing a password every time. You generate a key pair — a private key (stays on your computer) and a public key (goes on the remote machine).

- Generate an SSH key pair (on your local machine) :
  - `ssh-keygen -t ed25519 -C "your@email.com"`
  - Press Enter to accept the default location (`~/.ssh/id_ed25519`)
  - Optionally set a passphrase
- Copy your public key to the remote machine : `ssh-copy-id username@ip_address`
- Connect without a password : `ssh username@ip_address`
- View your public key : `cat ~/.ssh/id_ed25519.pub`

---

## Transferring Files via SSH

- Copy a file from local to remote : `scp /local/path/to/file.txt username@ip_address:/remote/path/`
- Copy a folder from local to remote : `scp -r /local/folder username@ip_address:/remote/path/`
- Copy a file from remote to local : `scp username@ip_address:/remote/path/file.txt /local/destination/`
- Copy a folder from remote to local : `scp -r username@ip_address:/remote/folder/ /local/destination/`

---

## Troubleshooting

- Connection refused :
  - Check that Remote Login is enabled (System Settings > Sharing)
  - Make sure you are on the same network
  - Double-check the IP address : `ipconfig getifaddr en0`
- Permission denied :
  - Check your username is correct : `whoami` on the target machine
  - If using SSH keys, check the key is properly set up : `ssh-copy-id username@ip`
- Host key verification failed :
  - Remove the old known host entry and try again : `ssh-keygen -R ip_address`
- Connection timeout :
  - The machine may be asleep — wake it up first
  - Check firewall settings on the target machine
