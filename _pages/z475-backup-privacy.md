---
title: "Z475 Backup · 隐私政策 / Privacy Policy"
permalink: /z475-backup-privacy/
layout: single
author_profile: false
read_time: false
comments: false
share: false
related: false
sitemap: false
excerpt: "Z475 Backup 使用 Google Drive API 为个人 Linux 服务器保存和恢复备份文件。"
---

*最后更新 / Last updated: 2026-10-18*

## 中文

### 1. 应用用途

**Z475 Backup** 是一个仅供本人使用的个人备份工具。它通过 [rclone](https://rclone.org/) 和 Google Drive API，将本人管理的 Lenovo Z475 Linux 服务器的配置文件、数据库导出文件及其他选定的备份文件存储到本人授权的 Google Drive 账号，并在需要时读取或恢复这些文件。本应用不面向公众提供账号注册或云盘托管服务。

### 2. 访问哪些 Google 用户数据？

经过 Google OAuth 授权后，rclone 可能获得对 Google Drive 文件及文件夹的读取、创建、修改和删除权限。当前配置使用 Google Drive 的较宽访问范围，**授权权限可能涵盖整个云端硬盘**，但本工具计划仅在专用备份文件夹（例如 `Lenovo-Z475-Backup`）中保存和管理备份。

备份文件可能包含服务器配置、API 凭据及数据库导出数据。备份文件**不额外进行客户端加密**，因此其保密性依赖于 Google 账号和 Google Drive 的安全措施。

### 3. 数据如何使用与存储？

- Google Drive 数据仅用于本人发起或安排的备份、校验、列举、删除旧备份及恢复操作。
- OAuth 凭据由 rclone 保存在本人管理的设备上，用于保持授权连接。
- 备份文件保存在本人 Google Drive 账号内；不会为本应用另行建立公开的数据收集网站或商业数据分析服务。
- 不出售 Google 用户数据，不将其用于广告或用户画像，也不主动向其他个人或商业第三方分享备份文件。Google 作为云存储提供商会按照其适用条款处理其服务中的数据。

### 4. 保留、删除与撤销授权

备份数据会保留到本人手动删除，或自动备份清理策略将其删除为止。本人可以随时在 Google Drive 中删除相关备份文件夹，也可以在 [Google 账号的第三方应用连接设置](https://myaccount.google.com/connections) 中撤销 Z475 Backup 的访问权限。撤销授权会阻止后续 API 访问，但不会自动删除此前已经保存在 Drive 中的文件；如需删除，应另外手动移除。

---

## English

### 1. Purpose

**Z475 Backup** is a personal-use backup setup operated by its owner. It uses [rclone](https://rclone.org/) and the Google Drive API to store and restore selected backup files from the owner's Lenovo Z475 Linux server in the owner's authorized Google Drive account. It is not a public file-hosting or account-registration service.

### 2. Google user data accessed

With the owner's OAuth consent, rclone may read, create, modify, and delete Google Drive files and folders. The currently configured Drive scope is broad and **may permit access to the entire Drive**, although the intended use is limited to a dedicated backup folder such as `Lenovo-Z475-Backup`.

Backups may include server configuration, API credentials, and database exports. **The backup archives are not additionally client-side encrypted.** Their confidentiality therefore depends on Google account and Google Drive security.

### 3. Use, storage, and sharing

- Drive access is used only for owner-initiated or scheduled backup, verification, listing, retention cleanup, and restore operations.
- rclone stores OAuth credentials on a device controlled by the owner to maintain the authorized connection.
- Backup files remain in the owner's Google Drive account. This setup does not operate a separate public data-collection or commercial analytics service.
- Google user data is not sold, used for advertising or profiling, or intentionally shared with unrelated individuals or commercial third parties. Google processes data stored in Google Drive under its applicable terms.

### 4. Retention, deletion, and revocation

Files are retained until the owner deletes them or a configured backup-retention policy removes them. The owner can delete backup files directly in Google Drive and can revoke the app's access through [Google Account connections](https://myaccount.google.com/connections). Revocation stops future authorized access but does not itself delete files already stored in Drive.

