# Overview - Systemd misconfiguration
This repository deploys a realistic, hardened Redis deployment using Ansible on an Ubuntu system.

The critical vulnerability lies in a misconfigured systemd timer which creates a backup of the redis config in a universally-accessible location, allowing any user to read the configuration and steal credentials which are stored in plaintext in the Redis config. The timer-run script also does not perform file validation, making it vulnerable to symlink attacks

# Secure Redis deployment

The environment deploys a production level configuration of Redis. It follows many security best practices, such as:
- Binding to localhost only
- Protected-mode is enabled
- Detailed ACL configuration with varying levels of privilege (always requires authentication)
- Mostly safe passwords
- Renamed dangerous commands for obfuscation
- Dangerous commands enabled only for administrators
- Systemd-managed service
- Two layers of persistence
- Disabled default user

Despite these protections, the vulnerability in systemd timer introduces a privilege escalation path for local users that circumvents most of these layers

# Basic vulnerability architeture
Compromised Local User (Low privilege)  
   │  
   ▼  
systemd timer  
    │  
    ▼  
Insecure backup script  
    │  
    ▼  
Find destination at /tmp/redis/backups (world-readable)  
    │  
    ▼  
Redis configuration exposure  
   │  
   ▼  
ACL credential theft  
   │  
   ▼  
Redis admin access  
   │  
   ▼  
CONFIG abuse → Code execution  
   │  
   ▼  
Privilege escalation to redis user  
   │  
   ▼  
Replace backup file with symlink  
   │  
   ▼
Script Runs  
   │    
   ▼  
Overwrite any file on the system

# Systemd Timer Implementation
The automation creates:
- World-writable directories under /tmp
- A root-owned backup script
- A systemd service for the running the script
- A recurring systemd timer

The timer executes this script:  

    /usr/local/bin/redis-backup.sh
# Security Issues in the Script
The script does not:
- Check if the destination file exists
- Verify ownership
- Check for symlinks
- Use safe file creation flags
- Use atomic writes
- Restrict directory permissions

This is what allows an attacker to abuse symlinks by deleting redis.conf.bak and replacing it with a symlink to any system file. Once the timer runs again, the script will overwrite the symlink target. Because the script is owned by root (which is completely normal for a systemd process), it has the ability to overwrite any file.

There are many possibilities, most of which are destructive (e.g overwriting kernel files). In a competition scenario, an attacker can use this to overwrite critical service files without breaking the box. For example, if the system is running Apache they could overwrite apache configs that they would otherwise not have access to as a compromised, low privilege user
