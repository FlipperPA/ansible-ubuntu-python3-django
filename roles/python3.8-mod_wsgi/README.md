# Install mod_wsgi for Apache

This role installs `mod_wsgi` for Apache, which will serve Django and Flask WSGI applications.

## Building a New `.so` File

To build an `.so` file for a new version of Python, install the new Python, and then use pip to build the so:

```bash
python3.x -m venv mod_wsgi_venv
. mod_wsgi_venv/bin/activate
pip install mod_wsgi
```

The `.so` file will be located at:
`mod_wsgi_venv/lib/python3.x/site-packages/mod_wsgi/server/mod_wsgi-py3x.cpython-3x-x86_64-linux-gnu.so`
