# Jellyfin Install (Docker, 2026)

Your own Netflix for the movies you already have: Jellyfin on any Debian or Ubuntu machine, on your TV and phone,
with Intel GPU transcoding, a kids profile, and watching away from home through Tailscale. Free, no account.

Video: *Plex Wants $750. Here's the Free Replacement (Jellyfin Setup 2026)* on [@kadirrangwala](https://www.youtube.com/@kadirrangwala)

Tested on Debian 13 with Jellyfin 12.4 and Docker 29.9.

---

## 1. Install Docker

Run these as root. If your server blocks root logins over SSH, log in as your normal user, then `sudo -i`
(or `su -` on Debian without sudo).

```bash
apt update && apt upgrade -y
apt install -y curl
curl -fsSL https://get.docker.com | sh
docker --version
```

## 2. Movies: one folder per movie, title and year

```
/srv/media/movies/
├── Elephants Dream (2006)/Elephants Dream (2006).mov
├── Sintel (2010)/Sintel (2010).mkv
└── Tears of Steel (2012)/Tears of Steel (2012).mov
```

Name each folder and file `Title (Year)` and Jellyfin finds the right poster and details for every movie.

Make the folder and give it to your normal (non-root) user, so you can copy movies into it from your PC:

```bash
mkdir -p /srv/media/movies
chown YOURUSER: /srv/media/movies     # your username, not "kadir"
ls -ld /srv/media/movies
```

Copy the movies over from your PC:

- **Windows:** [WinSCP](https://winscp.net) (free): **Download Now** → the green download button → run the
  installer with the default options. Then open it. New site: SFTP, host = your server's IP, your normal username and
  password → Login → Accept the host key the first time. Right side: go to `/srv/media/movies`. Left side: select
  your movie folders → **Upload** → OK.
- **Mac:** [Cyberduck](https://cyberduck.io) (free), SFTP, same login.

## 3. Run Jellyfin

```bash
mkdir -p /opt/jellyfin && cd /opt/jellyfin
ls /dev/dri          # files here = you have a GPU Jellyfin can use
nano compose.yaml
```

[`compose.yaml`](compose.yaml):

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /srv/media:/media:ro
    devices:
      - /dev/dri:/dev/dri
```

No `/dev/dri`? Delete the last two lines.

```bash
docker compose up -d
```

Open `http://<server-ip>:8096` and follow the wizard. Library: content type **Movies**, folder **/media/movies**.
Leave **Allow remote connections** on.

## 4. TV and phone

- Install the Jellyfin app from your TV's app store (Android TV / Google TV, Fire TV, Roku, LG, Samsung) or your
  phone's. On the same network it finds the server by itself.
- TV login without typing: choose **Quick Connect** on the TV, then on your computer: user menu → **Quick Connect** → type the code → **Authorize**.

## 5. GPU transcoding (Intel)

Dashboard → Playback → Transcoding → Hardware acceleration: **Intel QuickSync (QSV)** → tick **HEVC** → Save.
Check it:

1. Play a movie in the browser, player settings (gear) → **Quality** → something below the file's bitrate
   (I used 8 Mbps). Gear → **Playback Info** shows *Transcoding* and the reasons.
2. Dashboard → **Logs** → open the newest `FFmpeg.Transcode` log → Ctrl+F `h264_qsv`. If it's there, the GPU did
   the work. `libx264` instead means it's running on the CPU.

On my i5-10500T one transcode took about 62% of one CPU core in software, about 8% with QuickSync.

## 6. Kids profile

Dashboard → Users → **+** → name `Kids` → tick **Enable access to all libraries** (new users get none by default)
→ Save → **Parental Control**: maximum rating **PG**, block unrated **Movies** → Save.

Sign in as Kids: only PG-and-under movies with a rating show up.

## 7. Watch from anywhere (free, no port forwarding)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up          # open the link and sign in
tailscale ip -4       # your server's 100.x.y.z address
```

On your phone: install the Tailscale app → **Log in** with the same account → **Connect**. Then in the Jellyfin
app, Connect to Server → `100.x.y.z:8096` → sign in. Works on mobile data, no router changes.

## Updating later

```bash
cd /opt/jellyfin && docker compose pull && docker compose up -d
```

---

Demo films in the video: *Sintel*, *Tears of Steel*, *Elephants Dream* © Blender Foundation, released under Creative Commons Attribution licences, [studio.blender.org/films](https://studio.blender.org/films/).
