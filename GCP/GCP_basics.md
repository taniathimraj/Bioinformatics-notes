# GCP Basics

## VM Management

- List all VMs : `gcloud compute instances list`
- Start a VM : `gcloud compute instances start VM_NAME --zone ZONE`
- Stop a VM : `gcloud compute instances stop VM_NAME --zone ZONE`
  - Always stop your VM when not in use to avoid unnecessary charges
- SSH into a VM : `gcloud compute ssh USERNAME@VM_NAME --zone ZONE`
- Check VM status : `gcloud compute instances describe VM_NAME --zone ZONE`

## Basic Commands

- To check storage space: `du -sh /path/to/folder`
- To check which files are taking up space : `du -sh ~/*`
- To check full system space occupation : `sudo du -sh /* | sort -h`
- To run a script without interruptions and in the background: `nohup /path/to/script > /path/to/output/script.log 2>&1 &`
  - `nohup` = keeps running even if you disconnect
  - `2>&1` = captures both stdout and stderr into the log
  - `&` = runs in background
- To check if the script is running: `tail -f /path/to/output/script.log`
- To check the last 30 lines of a log : `tail -n 30 /path/to/logfile.log`
- To check if there were any errors recorded in the log file : `grep -E "error|fail|killed|terminated" /path/to/logfile.log`
- To check running processes : `ps aux | grep script_name`

## Storage and File Transfer

- To download data from the GCP to the computer :
  - Copy the files to a bucket in VM: `gsutil -m cp -r ~/path/to/folder gs://YOUR_BUCKET_NAME/`
  - In the local computer terminal : `gsutil -m cp -r gs://YOUR_BUCKET_NAME/folder ~/local/destination/`
- Upload folder to the GCP : `scp -i ~/.ssh/google_compute_engine -r foldername USERNAME@EXTERNAL_IP:~/`
- Upload folder to bucket : `gsutil -m cp -r /path/to/folder gs://YOUR_BUCKET_NAME/`
- Verify the upload : `gsutil ls gs://YOUR_BUCKET_NAME/foldername/`
- Retrieve the folder later : `gsutil -m cp -r gs://YOUR_BUCKET_NAME/foldername ~/`
- Download file directly from VM to local computer : `gcloud compute scp USERNAME@VM_NAME:/remote/path/to/file /local/path --zone ZONE`

## tmux (Keep Sessions Alive When Disconnected from VM)

- New tmux session : `tmux new -s session_name`
- Detach from session (keeps it running in background) : `Ctrl + B`, then `D`
- List all active sessions : `tmux ls`
- Reattach to an existing session : `tmux attach -t session_name`
- Kill a session : `tmux kill-session -t session_name`
- Exit and close a session : `exit`

## BAM File Checks

- To check integrity of the bam files : `samtools quickcheck -v file.bam`
- To check the log files: `tail -n 30 /path/to/logfile.log`
- To check if there were any errors recorded in the log file : `grep -E "error|fail|killed|terminated" /path/to/logfile.log`

## File Operations

- To find if a file is there : `find ~/ -type f -name "*filename*.extension"`
- To monitor how file sizes change, refresh every 10 sec: `watch -n 10 'ls -lh /path/to/folder/*.bam'`
- To copy files from a folder to current directory: `cp /path/to/folder/*.bam* .`
- To move files with a specific prefix : `mv /path/to/folder/PREFIX* .`
- To add a prefix to all files in a folder : `for f in *; do mv "$f" "prefix_${f}"; done`

## Billing and Cost Tips

- Check current billing : GCP Console > Billing (console.cloud.google.com/billing)
- Always stop VMs when not in use
- Use preemptible VMs for non-critical long jobs (much cheaper)
- Delete unused persistent disks and clean up old files in buckets (storage costs money)
- Set up billing alerts in the console to get notified when you hit a spending threshold
