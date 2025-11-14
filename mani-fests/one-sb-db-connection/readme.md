kubectl -n insurance-demo create secret generic db-credentials \
  --from-literal=DB_HOST="<DB_VM_IP>" \
  --from-literal=DB_PORT="<DB_PORT>" \
  --from-literal=DB_NAME="<DB_NAME>" \
  --from-literal=DB_USER="<DB_USER>" \
  --from-literal=DB_PASSWORD="<DB_PASSWORD>" \
  --dry-run=client -o yaml | kubectl apply -f -