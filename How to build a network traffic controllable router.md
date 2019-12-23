How to build a network traffic controllable router
==================

## I. Hardwares and foundation build up
### Hardwares and OS
1. Computer: x86 CPU machine, 1 piece, installed OS: Ubuntu 18.04.2 LTS Desktop version
2. Network adapter: USB_to_Ethernet Adapter, 2 pieces
3. Router: Wifi 802.11AC router which should support AP mode, 1 piece

### Foundation config
1. Turn off the computer's Wifi
2. Need link these components like this: [Wifi Router] WAN port <-> [USB_to_Ethernet Adapter #1] <-> [Computer] <-> [USB_to_Ethernet Adapter #2] <-> [WAN wired network port]. Please do not use the default Ethernet port of the computer, and make sure it is not used at all
3. In the Ubuntu Terminal, run `nm-connection-editor`. In the popped up window, find the [USB_to_Ethernet Adapter #1] and edit its config. Tap IPV4 tab, and change the 'method' to *Shared to other computers*
4. Use a phone to connect with [Wifi router] using wifi, then try visiting some websites to see if the network is reachable


## II. Config and run Facebook Augmented Traffic Control
### Clone code
1. Make project root directory *fb-augmented-traffic-control*
2. Clone https://github.com/yn295636/augmented-traffic-control into subdir *src*

### Install dependencies
1. Create python 2.7 virtual-env into subdir *venv*
2. Activate the virtual-env and run `pip install src/atc/*`

### Config atcui with django
1. Activate the virtual-env
2. Run `django-admin startproject atcui`
3. Edit *atcui/atcui/settings.py*, change the config like below lines
```python
INSTALLED_APPS = (
    ...
    # Django ATC API
    'rest_framework',
    'atc_api',
    # Django ATC Demo UI
    'bootstrap_themes',
    'django_static_jquery',
    'atc_demo_ui',
    # Django ATC Profile Storage
    'atc_profile_storage',
)

```
4. Edit *atcui/atcui/urls.py*, change the config like below lines
```python
...
from django.views.generic.base import RedirectView
from django.conf.urls import include

urlpatterns = [
    ...
    # Django ATC API
    url(r'^api/v1/', include('atc_api.urls')),
    # Django ATC Demo UI
    url(r'^atc_demo_ui/', include('atc_demo_ui.urls')),
    # Django ATC profile storage
    url(r'^api/v1/profiles/', include('atc_profile_storage.urls')),
    url(r'^$', RedirectView.as_view(url='/atc_demo_ui/', permanent=False)),
]
```
5. Update django DB
```bash
cd atcui
python manage.py migrate
```

### Create service start shell
1. *start_atcd.sh*
```bash
cd [PATH_TO_PROJECT_ROOT]
source venv/bin/activate
sudo atcd --atcd-wan NETWORK_INTERFACE_ID_TO_INTERNET --atcd-lan NETWORK_INTERFACE_ID_AS_LAN
```
2. *start_atcui.sh*
```bash
cd [PATH_TO_PROJECT_ROOT]
source venv/bin/activate
cd atcui
python manage.py runserver 0.0.0.0:8000
```
3. Add executable permission for them
```
chmod a+x start_atcd.sh start_atcui.sh
```

### Config supervisor for the services(use Ubuntu as example)
1. Install supervisor under python system library. `sudo apt-get install supervisor`
2. Edit */etc/supervisor/supervisord.conf*, change config like below lines
```
...
[inet_http_server]
port=[WAN_IP]:9001
username=admin
password=[ADMIN_PASSWORD]
...
...
[include]
files = /etc/supervisor/conf.d/*.conf
...
```
3. Create supervisor service config file */etc/supervisor/conf.d/atcui.conf*
```
[program:atcd]
command=[PATH_TO_PROJECT_ROOT]/start_atcd.sh
autostart=true
autorestart=true
startretries=3
stopasgroup=true
stopsignal=QUIT
redirect_stderr=true
stdout_logfile=/var/log/atc/atcd.log
```
4. Create supervisor service config file */etc/supervisor/conf.d/atcd.conf*
```
[program:atcui]
command=[PATH_TO_PROJECT_ROOT]/start_atcui.sh
autostart=true
autorestart=true
startretries=3
stopasgroup=true
stopsignal=QUIT
redirect_stderr=true
stdout_logfile=/var/log/atc/atcui.log
```
5. Make folder *atc* under */var/log* with sudo
6. Reload supervisor
```bash
sudo supervisorctl reload
sudo supervisorctl start atcd atcui
```

### Import network profile presets
```bash
cd [PATH_TO_PROJECT_ROOT]/src/utils
sh restore-profiles.sh localhost:8000
```
