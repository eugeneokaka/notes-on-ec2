# Notes Library

A single-page notes portal that lists course documents stored in an Amazon S3 bucket and lets visitors preview them directly in the browser.

Live site is a static `index.html` served by Apache (`httpd`) on an Amazon EC2 instance. The documents themselves stay in S3 and are only linked, never copied to the server.

## Features

- **One-file app** – the entire website is `index.html` (HTML + CSS + JS, no build step, no dependencies).
- **Automatic titles** – document titles are derived from the S3 file names (UUIDs, `_`, `+`, `- kabarak university`, and `(1)` suffixes are cleaned up, then title-cased).
- **Semester grouping** – notes are shuffled into "Semester 1" and "Semester 2" sections on every page load (or when the **Shuffle** button is pressed).
- **Search** – filters cards by title or raw file name as you type.
- **In-page preview**
  - `.pdf` files render natively in an `<iframe>` (S3 serves them as `application/pdf`).
  - `.docx` files render through the Microsoft Office Online viewer (`view.officeapps.live.com`), because browsers cannot display Word documents natively.
- **Way back out** – a highlighted **← Back** button, an **×** close button, `Esc`, and the browser's own Back button all return to the library.
- **Preview toolbar** – shows the document title plus **Open in new tab** and **Download** links.

## How it works

1. `urls.txt` holds the S3 object URLs (one per line).
2. The URLs are embedded in the `URLS` array inside `index.html` (a static page cannot read `urls.txt` when opened via `file://` or without a fetch path).
3. On load the script:
   - extracts and prettifies each title from its URL,
   - shuffles the list,
   - splits it in half into Semester 1 / Semester 2,
   - renders each item as a card with a `PDF` / `DOCX` badge.
4. Clicking a card opens the overlay viewer:
   - PDF → direct S3 URL in the iframe,
   - DOCX → `https://view.officeapps.live.com/op/embed.aspx?src=<encoded S3 URL>`.
5. Opening a preview pushes a browser history entry so the Back button closes it.

## Repository files

| File         | Purpose                                              |
| ------------ | ---------------------------------------------------- |
| `index.html` | The whole application (markup, styles, logic).       |
| `urls.txt`   | Source list of S3 document URLs.                     |
| `README.md`  | This document.                                       |

## EC2 deployment (Amazon Linux 2023)

The provisioning script:

```bash
#!/bin/bash
# 1. Update packages
sudo dnf update -y

# 2. Install Apache
sudo dnf install httpd -y

# 3. Create the website directory
sudo mkdir -p /var/www/html

# 4. Write the site (index.html) into place
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
...the full index.html...
EOF

# 5. Set permissions
sudo chmod 644 /var/www/html/index.html

# 6. Enable and start Apache
sudo systemctl enable httpd
sudo systemctl start httpd

# 7. Check Apache status
sudo systemctl status httpd --no-pager
```

After it finishes, open the instance's public IPv4/DNS (or the attached Elastic IP) in a browser. Make sure the security group allows inbound HTTP on port 80.

To update the site later, rewrite `/var/www/html/index.html` (or `scp` the file there) and reload:

```bash
sudo systemctl reload httpd
```

## Adding or changing notes

1. Upload the file to the `bucketfornotes` bucket (region `eu-north-1`) and make sure the object is publicly readable.
2. Add its URL to `URLS` in `index.html`:

```js
const URLS = [
  "https://bucketfornotes.s3.eu-north-1.amazonaws.com/ACCT+110++BL+FOUNDATIONS+OF+ACCOUNTING+-+kabarak+university.pdf",
  // ...
];
```

3. Copy the updated file to the server and reload Apache.

Titles are generated from the file name, so keep it descriptive (e.g. `MATH+113+CALCULUS+I+-+kabarak+university.docx` becomes *Math 113 Calculus I*).

## Bucket requirements

- Objects must be publicly readable (direct links with no presigned URLs or authentication).
- The bucket does **not** need CORS for the PDF iframe; the Office Online viewer fetches the file server-side and only requires the URL to be publicly accessible.
- PDFs must be served with `Content-Type: application/pdf`. S3 does this automatically, but verify if you re-upload with custom metadata.

## Limitations

- Semester placement is **random** — file names do not contain semester information, so the split into Semester 1 / Semester 2 changes on each load. To make it deterministic, replace the `shuffle(...)`/split logic with a fixed assignment.
- No authentication; anyone with the URL can view the listed notes.
- DOCX preview depends on Microsoft's free Office Online viewer being reachable and the file being publicly accessible. If it fails, use **Download** or **Open in new tab**.
- The document list is compiled into `index.html`; editing `urls.txt` alone does not update a deployed site.
