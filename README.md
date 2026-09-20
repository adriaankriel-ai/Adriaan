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
name: Verify Through SSH Tunnel

on:
  workflow_dispatch:

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    env:
      DB_HOST: 127.0.0.1
      DB_PORT: "5432"
      BASTION_HOST: bastion.example.com
      BASTION_USER: deploy
      PRIVATE_DB_HOST: private-db.example.internal
      PRIVATE_DB_PORT: "5432"

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Verify through secure SSH tunnel
        shell: bash
        env:
          SSH_PRIVATE_KEY: ${{ secrets.BASTION_SSH_KEY }}
          BASTION_HOST_KEY: ${{ secrets.BASTION_HOST_PUBLIC_KEY }}
        run: |
          set -uo pipefail

          SSH_DIR="$RUNNER_TEMP/ssh"
          SSH_KEY="$SSH_DIR/id_ed25519"
          KNOWN_HOSTS="$SSH_DIR/known_hosts"
          CONTROL_SOCKET="$RUNNER_TEMP/bastion-tunnel.sock"

          mkdir -p "$SSH_DIR"
          chmod 700 "$SSH_DIR"

          # Write the private key without exposing it in logs.
          printf '%s\n' "$SSH_PRIVATE_KEY" > "$SSH_KEY"
          chmod 600 "$SSH_KEY"

          # BASTION_HOST_PUBLIC_KEY must be a complete known_hosts line, for example:
          # bastion.example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
          printf '%s\n' "$BASTION_HOST_KEY" > "$KNOWN_HOSTS"
          chmod 600 "$KNOWN_HOSTS"

          cleanup() {
            if [[ -S "$CONTROL_SOCKET" ]]; then
              ssh \
                -i "$SSH_KEY" \
                -S "$CONTROL_SOCKET" \
                -o BatchMode=yes \
                -o StrictHostKeyChecking=yes \
                -o UserKnownHostsFile="$KNOWN_HOSTS" \
                -O exit \
                "$BASTION_USER@$BASTION_HOST" \
                >/dev/null 2>&1 || true
            fi

            rm -f "$SSH_KEY" "$KNOWN_HOSTS" "$CONTROL_SOCKET"
          }

          trap cleanup EXIT

          # One SSH command creates the background tunnel.
          ssh \
            -i "$SSH_KEY" \
            -o IdentitiesOnly=yes \
            -o BatchMode=yes \
            -o ExitOnForwardFailure=yes \
            -o StrictHostKeyChecking=yes \
            -o UserKnownHostsFile="$KNOWN_HOSTS" \
            -M \
            -S "$CONTROL_SOCKET" \
            -f -N \
            -L "$DB_PORT:$PRIVATE_DB_HOST:$PRIVATE_DB_PORT" \
            "$BASTION_USER@$BASTION_HOST"

          echo "SSH tunnel established: $DB_HOST:$DB_PORT"

          # These scripts are not currently present in the repository.
          # Add them before enabling this section.
          if [[ ! -x ./run_verify.sh ]]; then
            echo "::error::run_verify.sh is missing or not executable."
            exit 1
          fi

          if [[ ! -x ./verify_scenarios.sh ]]; then
            echo "::error::verify_scenarios.sh is missing or not executable."
            exit 1
          fi

          set +e
          ./run_verify.sh ./verify_scenarios.sh > "$RUNNER_TEMP/staging_verify.log" 2>&1
          verify_exit=$?
          set -e

          cat "$RUNNER_TEMP/staging_verify.log"
          echo "Verification finished with exit code: $verify_exit"
          exit "$verify_exit"