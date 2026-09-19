# Adriaan
git init git add . git commit -m "initial commit: wired auth form; unguard /login" git branch -M main  gh repo create adriaan-kriel-office --public --source=. --remote=origin --push
# GitHub Actions Step Example
- name: Open SSH Tunnel and Run Verify
  env:
    SSH_PRIVATE_KEY: ${{ secrets.BASTION_SSH_KEY }}
    # Tell your script to point to localhost instead of the real DB IP
    DB_HOST: "127.0.0.1" 
    DB_PORT: "5432" # Example for PostgreSQL
  run: |
    # 1. Setup SSH Key
    mkdir -p ~/.ssh
    echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa
    
    # 2. Open tunnel in background (Maps local port 5432 to remote database port 5432 via Bastion)
    ssh -f -N -L 5432:private-db-hostname-or-ip:5432 user@bastion-ip -o StrictHostKeyChecking=no
    
    # 3. Make scripts executable and run
    chmod +x run_verify.sh verify_scenarios.sh
    ./run_verify.sh verify_scenarios.sh > staging_verify_2025.log 2>&1 || verify_exit=$?
    
    verify_exit=${verify_exit:-0}
    echo "Verification finished with exit code: $verify_exit"
    exit $verify_exit
