# Renew Octavia client certificate trên Kolla-Ansible 2025.1

Tài liệu này dùng cho trường hợp Octavia Amphora trên Kolla-Ansible `2025.1` bị lỗi TLS do `client.cert-and-key.pem` hết hạn.

Ví dụ lỗi:

```text
requests.exceptions.SSLError: HTTPSConnectionPool(host='<amphora_ip>', port=9443)
Caused by SSLError(SSLError(1, '[SSL: SSLV3_ALERT_CERTIFICATE_EXPIRED] sslv3 alert certificate expired'))
```

Ví dụ cert đã hết hạn:

```text
notBefore=Apr 30 22:18:32 2025 GMT
notAfter=Apr 30 22:18:32 2026 GMT
```

## Nguyên tắc xử lý

Trong Kolla-Ansible, Octavia cần các TLS certificates riêng cho Amphora provider. Tài liệu Kolla-Ansible `2025.1` ghi rằng lệnh:

```bash
kolla-ansible octavia-certificates
```

sẽ tạo certificate/key dưới:

```text
/etc/kolla/config/octavia
```

Với Kolla-Ansible `stable/2025.1`, role `octavia-certificates` đặt mặc định:

```yaml
octavia_certs_server_ca_expiry: 3650
octavia_certs_client_ca_expiry: 3650
octavia_certs_client_expiry: 365
```

Nếu muốn tạo `client.cert-and-key.pem` có thời hạn dài hơn, ví dụ 5 năm, override biến sau trong `/etc/kolla/globals.yml` trước khi chạy lại `kolla-ansible octavia-certificates`:

```yaml
octavia_certs_client_expiry: 1825
```

Lưu ý: biến này chỉ ảnh hưởng tới **client certificate của Octavia controller/worker**. Nó không làm tăng thời hạn server certificate nằm bên trong amphora. Server certificate của amphora dùng cấu hình Octavia riêng, ví dụ `[certificates] cert_validity_time`.

Vì vậy, nếu chỉ `client.cert-and-key.pem` hết hạn còn `client_ca.cert.pem` và `server_ca.cert.pem` chưa hết hạn, hướng xử lý an toàn là:

```text
Renew client.cert-and-key.pem
Giữ nguyên client_ca.cert.pem
Giữ nguyên server_ca.cert.pem
Không rotate CA
Không failover amphora hàng loạt nếu verify mTLS thành công
```

Lý do: amphora cũ đã trust `client_ca` cũ. Nếu client cert mới vẫn được ký bởi cùng `client_ca`, amphora cũ vẫn có thể chấp nhận cert mới. Nếu rotate `client_ca` hoặc `server_ca`, trust chain thay đổi và thường phải failover/recreate amphora để đồng bộ lại certificate/trust.

## 1. Kiểm tra trước khi thay đổi

Chạy trên deploy node:

```bash
kolla-ansible -i <inventory> octavia-certificates --check-expiry 365
```

Kiểm tra ngày hết hạn và fingerprint hiện tại:

```bash
openssl x509 -in /etc/kolla/config/octavia/client_ca.cert.pem -noout -dates -subject -fingerprint -sha256
openssl x509 -in /etc/kolla/config/octavia/server_ca.cert.pem -noout -dates -subject -fingerprint -sha256
openssl x509 -in /etc/kolla/config/octavia/client.cert-and-key.pem -noout -dates -subject -fingerprint -sha256
```

Ghi nhận fingerprint của:

```text
client_ca.cert.pem
server_ca.cert.pem
```

Sau khi renew, hai fingerprint này phải không đổi.

## 2. Backup

```bash
BACKUP=/root/octavia-cert-backup-$(date +%F-%H%M%S)
mkdir -p $BACKUP

cp -a /etc/kolla/config/octavia $BACKUP/config-octavia
cp -a /etc/kolla/octavia-certificates $BACKUP/octavia-certificates

openssl x509 -in /etc/kolla/config/octavia/client_ca.cert.pem -noout -fingerprint -sha256 > $BACKUP/client_ca.before
openssl x509 -in /etc/kolla/config/octavia/server_ca.cert.pem -noout -fingerprint -sha256 > $BACKUP/server_ca.before
```

## 3. Renew chỉ client certificate

Xóa các file client cert/key/csr để Kolla-Ansible tạo lại phần client certificate, nhưng giữ nguyên CA:

```bash
cd /etc/kolla/octavia-certificates/client_ca

rm -f client.key.pem client.csr.pem client.cert.pem client.cert-and-key.pem
rm -f /etc/kolla/config/octavia/client.cert-and-key.pem
```

Cập nhật database của OpenSSL CA:

```bash
grep '^octavia_client_ca_password:' /etc/kolla/passwords.yml
```

Khi OpenSSL hỏi:

```text
Enter pass phrase for client_ca.key.pem:
```

nhập giá trị của `octavia_client_ca_password`. Đây là passphrase của `client_ca.key.pem`; không dùng `octavia_ca_password` cho bước này.

```bash
openssl ca -config ../openssl.cnf -name client_ca -updatedb
```

Lệnh `openssl ca -updatedb` cập nhật CA database, đánh dấu các certificate đã hết hạn thành expired. Lệnh này không tạo cert mới và không rotate CA.

Tạo lại Octavia certificates:

```bash
kolla-ansible -i <inventory> octavia-certificates
```

## 4. Verify sau khi tạo cert mới

Kiểm tra cert client mới:

```bash
openssl x509 -in /etc/kolla/config/octavia/client.cert-and-key.pem -noout -dates -subject -fingerprint -sha256
```

Kiểm tra `client_ca` và `server_ca` không đổi:

```bash
openssl x509 -in /etc/kolla/config/octavia/client_ca.cert.pem -noout -fingerprint -sha256 > /tmp/client_ca.after
openssl x509 -in /etc/kolla/config/octavia/server_ca.cert.pem -noout -fingerprint -sha256 > /tmp/server_ca.after

diff -u $BACKUP/client_ca.before /tmp/client_ca.after
diff -u $BACKUP/server_ca.before /tmp/server_ca.after
```

Hai lệnh `diff` không được có khác biệt. Nếu fingerprint khác, nghĩa là CA đã bị thay đổi, không nên tiếp tục apply production khi chưa đánh giá tác động lên amphora cũ.

Tách phần certificate ra khỏi file `client.cert-and-key.pem` rồi verify cert client được ký bởi `client_ca`:

```bash
awk '/BEGIN CERTIFICATE/{p=1} p{print} /END CERTIFICATE/{exit}' \
  /etc/kolla/config/octavia/client.cert-and-key.pem > /tmp/octavia-client.cert.pem

openssl verify -CAfile /etc/kolla/config/octavia/client_ca.cert.pem /tmp/octavia-client.cert.pem
```

Kết quả mong đợi:

```text
/tmp/octavia-client.cert.pem: OK
```

## 5. Apply vào Octavia containers

```bash
kolla-ansible -i <inventory> reconfigure --tags octavia
```

Nếu muốn chắc chắn các service Octavia load cert mới ngay:

```bash
docker restart octavia_worker octavia_health_manager octavia_housekeeping
```

Hoặc nếu môi trường dùng Podman:

```bash
podman restart octavia_worker octavia_health_manager octavia_housekeeping
```

## 6. Verify Octavia control được amphora cũ

Lấy danh sách amphora:

```bash
. /etc/kolla/admin-openrc.sh
openstack loadbalancer amphora list
```

Chọn một amphora cũ, lấy `lb_network_ip`, rồi test mTLS từ node có đường tới Octavia management network:

```bash
echo | openssl s_client \
  -connect <amphora_lb_network_ip>:9443 \
  -cert /etc/kolla/config/octavia/client.cert-and-key.pem \
  -key /etc/kolla/config/octavia/client.cert-and-key.pem \
  -CAfile /etc/kolla/config/octavia/server_ca.cert.pem \
  -verify_return_error \
  -brief
```

Kết quả mong đợi:

```text
Không còn certificate expired
Không còn SSLV3_ALERT_CERTIFICATE_EXPIRED
Handshake TLS thành công
```

Kiểm tra log Octavia sau khi apply:

```bash
docker logs --since 5m octavia_worker 2>&1 | egrep 'certificate expired|SSLV3_ALERT_CERTIFICATE_EXPIRED|AmpConnectionRetry'
docker logs --since 5m octavia_health_manager 2>&1 | egrep 'certificate expired|SSLV3_ALERT_CERTIFICATE_EXPIRED|AmpConnectionRetry'
```

Hoặc với Podman:

```bash
podman logs --since 5m octavia_worker 2>&1 | egrep 'certificate expired|SSLV3_ALERT_CERTIFICATE_EXPIRED|AmpConnectionRetry'
podman logs --since 5m octavia_health_manager 2>&1 | egrep 'certificate expired|SSLV3_ALERT_CERTIFICATE_EXPIRED|AmpConnectionRetry'
```

Nếu không còn log lỗi TLS expired và mTLS tới amphora cũ thành công, không cần failover các LB cũ.

## 7. Khi nào cần failover amphora

Không cần failover nếu tất cả điều kiện sau đúng:

```text
Chỉ client.cert-and-key.pem được tạo lại
client_ca.cert.pem không đổi fingerprint
server_ca.cert.pem không đổi fingerprint
openssl verify client cert với client_ca trả OK
openssl s_client tới amphora cũ qua port 9443 không còn lỗi expired
```

Cần đánh giá failover/recreate amphora nếu gặp một trong các trường hợp:

```text
client_ca.cert.pem bị đổi
server_ca.cert.pem bị đổi
server certificate bên trong amphora cũng hết hạn
openssl s_client tới amphora cũ vẫn báo certificate expired sau khi renew client cert
Octavia worker vẫn báo AmpConnectionRetry do TLS expired sau khi reconfigure/restart
```

Với LB topology `SINGLE`, failover có downtime. Với `ACTIVE_STANDBY`, vẫn nên thực hiện theo từng LB và có cửa sổ bảo trì nếu là production.

## 8. Failover từng amphora

Nếu một load balancer dùng topology `ACTIVE_STANDBY`, có thể failover từng amphora thay vì failover cả load balancer một lần. Cách này giúp kiểm soát rủi ro tốt hơn khi cần xử lý nhiều LB.

Lấy danh sách amphora của một LB:

```bash
openstack loadbalancer amphora list --loadbalancer <lb-id>
```

Ghi lại `id`, `role`, `status` và `lb_network_ip` của từng amphora:

```text
BACKUP
MASTER
```

Thứ tự khuyến nghị với `ACTIVE_STANDBY`:

```text
1. Failover BACKUP trước.
2. Chờ amphora mới lên ổn định.
3. Verify mTLS tới amphora mới.
4. Sau đó mới failover MASTER.
```

Failover amphora `BACKUP`:

```bash
openstack loadbalancer amphora failover --wait <backup-amphora-id>
```

Kiểm tra lại amphora của LB:

```bash
openstack loadbalancer amphora list --loadbalancer <lb-id>
```

Lấy `lb_network_ip` của amphora mới, rồi verify mTLS:

```bash
echo | openssl s_client \
  -connect <new_backup_amphora_lb_network_ip>:9443 \
  -cert /etc/kolla/config/octavia/client.cert-and-key.pem \
  -key /etc/kolla/config/octavia/client.cert-and-key.pem \
  -CAfile /etc/kolla/config/octavia/server_ca.cert.pem \
  -verify_return_error \
  -brief
```

Nếu path `server_ca.cert.pem` không có ở `/etc/kolla/config/octavia`, dùng CA trong workdir:

```bash
echo | openssl s_client \
  -connect <new_backup_amphora_lb_network_ip>:9443 \
  -cert /etc/kolla/config/octavia/client.cert-and-key.pem \
  -key /etc/kolla/config/octavia/client.cert-and-key.pem \
  -CAfile /etc/kolla/octavia-certificates/server_ca/server_ca.cert.pem \
  -verify_return_error \
  -brief
```

Khi `BACKUP` mới đã ổn, tiếp tục failover `MASTER`:

```bash
openstack loadbalancer amphora failover --wait <master-amphora-id>
```

Sau đó kiểm tra lại:

```bash
openstack loadbalancer show <lb-id>
openstack loadbalancer amphora list --loadbalancer <lb-id>
```

Kiểm tra log Octavia:

```bash
docker logs --since 10m octavia_worker 2>&1 | egrep 'certificate expired|certificate verify failed|AmpConnectionRetry|ComputeWaitTimeout'
docker logs --since 10m octavia_health_manager 2>&1 | egrep 'certificate expired|certificate verify failed|AmpConnectionRetry'
```

Với Podman:

```bash
podman logs --since 10m octavia_worker 2>&1 | egrep 'certificate expired|certificate verify failed|AmpConnectionRetry|ComputeWaitTimeout'
podman logs --since 10m octavia_health_manager 2>&1 | egrep 'certificate expired|certificate verify failed|AmpConnectionRetry'
```

Lưu ý:

```text
Không chạy failover quá nhiều amphora cùng lúc.
Nên làm canary 1 LB trước.
Sau đó chạy theo batch nhỏ, ví dụ 3-5 LB/lượt.
Với LB topology SINGLE, failover amphora gần như tương đương thay toàn bộ dataplane và có downtime.
```

## 9. Rollback

Nếu cần quay lại backup:

```bash
cp -a $BACKUP/config-octavia/* /etc/kolla/config/octavia/
cp -a $BACKUP/octavia-certificates/* /etc/kolla/octavia-certificates/

kolla-ansible -i <inventory> reconfigure --tags octavia
```

Sau rollback, kiểm tra lại ngày hết hạn và log. Nếu rollback về cert đã hết hạn, lỗi TLS expired sẽ quay lại; rollback chỉ nên dùng để phục hồi trạng thái file khi quá trình renew bị sai.

## Nguồn tham khảo

- Kolla-Ansible 2025.1 Octavia documentation: https://docs.openstack.org/kolla-ansible/2025.1/reference/networking/octavia.html
- Kolla-Ansible `stable/2025.1` role `octavia-certificates`: https://opendev.org/openstack/kolla-ansible/src/branch/stable/2025.1/ansible/roles/octavia-certificates
- Kolla-Ansible `stable/2025.1` defaults for Octavia certificate expiry: https://opendev.org/openstack/kolla-ansible/src/branch/stable/2025.1/ansible/roles/octavia-certificates/defaults/main.yml
- Kolla-Ansible `stable/2025.1` client certificate generation task: https://opendev.org/openstack/kolla-ansible/src/branch/stable/2025.1/ansible/roles/octavia-certificates/tasks/client_cert.yml
- OpenStackClient Octavia CLI, including amphora failover: https://docs.openstack.org/python-octaviaclient/2025.1/cli/index.html
- OpenSSL `ca` manual, including `-updatedb`: https://docs.openssl.org/1.1.1/man1/ca/
