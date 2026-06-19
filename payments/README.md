# Tenant `payments` — phòng riêng cô lập cho team B

Đón team thứ hai vào cụm dùng chung, cấp "phòng riêng": tự quản phần mình,
không đụng đồ team `demo`, 2 team không gọi qua lại. Guardrail bảo mật cũ
tự áp cho team mới — **không viết luật mới**.

## Thành phần (đều qua GitOps)

| File | Vai trò |
|------|---------|
| `namespace.yaml` | ns `payments` + label opt-in guardrail cosign |
| `rbac.yaml` | Role + RoleBinding `payments-dev` — bó trong ns, chỉ workload |
| `quota.yaml` | ResourceQuota (ngân sách) + LimitRange (default limits) |
| `networkpolicy.yaml` | default-deny ingress + egress chỉ cùng-ns & DNS |
| `app.yaml` | app team B (image đã ký + đủ limits) |

## ① Vì sao guardrail cũ TỰ áp cho team B mà không cần viết luật mới?

Các guardrail là **admission policy ở phạm vi cluster**, không gắn chết vào ns `demo`:

- **Gatekeeper** (`ConstraintTemplate` + `Constraint`) và **Sigstore**
  (`ClusterImagePolicy`) đều là tài nguyên **cluster-scoped**. Logic luật
  (Rego, chính sách chữ ký) viết **một lần**.
- Việc enforce do **admission webhook** chặn ở API server — nó soi *mọi*
  pod được tạo, ở *bất kỳ* namespace nào đã opt-in.
- Team B chỉ cần **vào phạm vi**: ns `payments` được thêm vào `match.namespaces`
  của Constraint, và gắn label `policy.sigstore.dev/include=true` để bật verify
  chữ ký. **Không** thêm `ConstraintTemplate`/Rego/`ClusterImagePolicy` mới.

→ Hệ quả: app team B muốn chạy phải **đã ký + đủ limits + tag pin + không root**…
y như team A. Một manifest vi phạm trong `payments` bị **chính constraint cũ** chặn.

## ② Role/RoleBinding khác ClusterRoleBinding thế nào để giữ cô lập?

| | Phạm vi | Hệ quả với cô lập |
|---|---|---|
| **Role + RoleBinding** | **1 namespace** | quyền chỉ có hiệu lực trong ns được bind |
| **ClusterRole + ClusterRoleBinding** | **toàn cụm** | quyền áp ở *mọi* namespace |

`payments-dev` được cấp bằng **Role + RoleBinding trong ns `payments`**:
- Tạo/sửa deployment trong `payments` → **được**.
- Cùng động tác đó sang `demo` → **bị từ chối** (không có binding nào ở `demo`).

Nếu lỡ dùng **ClusterRoleBinding**, quyền sẽ "với" sang `demo` → phá cô lập.
Vì vậy tenant phải dùng RoleBinding bó trong ns.

> Ngoài ra Role cố ý **không** cấp `secrets` (đọc bí mật) và `roles/rolebindings`
> (tự nâng quyền) — least-privilege.

## Nghiệm thu nhanh

```bash
# 1. RBAC bó ns
kubectl auth can-i create deployment --as=payments-dev -n payments   # yes
kubectl auth can-i create deployment --as=payments-dev -n demo       # no
kubectl auth can-i get secrets       --as=payments-dev -n payments   # no
kubectl auth can-i update rolebindings --as=payments-dev -n payments # no

# 2. Quota + LimitRange
#   - pod xin RAM vượt quota -> Forbidden (exceeded quota)
#   - pod không khai limits  -> vẫn chạy (LimitRange gán default)

# 3. NetworkPolicy (cần Calico)
#   - pod payments curl service api.demo -> timeout/chặn

# 4. Guardrail kế thừa
#   - app payments-api (đã ký 0.0.3) -> chạy xanh
#   - deploy manifest vi phạm trong payments -> bị constraint cũ chặn
```
