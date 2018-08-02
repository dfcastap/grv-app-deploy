# grv-app-deploy

This repo is a self-contained copy of the working app.

It works locally, the aim is to deploy it on Amazon EC2

## Procedure

This procedure is based on [this blog](http://bit.ly/2LJUONn) and its help is gratefully acknowledged

1. Sign in to AWS services
2. Launch EC2 instance using `Canonical, Ubuntu, 16.04 LTS, amd64 xenial image build on 2018-06-27`
3. Connect to the instance using SSH where permissions on `key.pem` were set with
    `$ chmod 400 key.pem`
4. Connect with `ssh -i ~/.ssh/grv-app-deploy.pem ubuntu@ec2-54-84-76-221.compute-1.amazonaws.com`
5. Update apt-get
    `$ sudo apt-get update`
6. Check no Python with `$ python -V` output: 
    The program 'python' can be found in the following packages:
    * python-minimal
    * python3
7. Install python 3.6 with:
    * `sudo add-apt-repository ppa:deadsnakes/ppa`
    * `sudo apt-get update`
    * `sudo apt-get install python3.6 python3.6-dev`
9. Install pip3 with `$ sudo apt-get install python3-pip`
10. Get virtualenv with `pip3 install virtualenv`
11. If `locale.Error: unsupported locale setting` error, run `export LC_ALL=C` from [here](http://bit.ly/2NK40lv)
12. Then run `pip3 install virtualenv` again
13. Create the project folder or clone the project from your Git repository
14. Set up the virtual environment.
    `$ cd cloned_project`
    `$ virtualenv py36 --python=python3.6` to create a Python 3.6 environment - Python 3.6 is required for altair
15. Activate the environment (use deactivate to exit the environment).
    `$ . py366/bin/activate`
16. Check python version with `$ python -V` output: `Python 3.6.6`
17. Install Flask and other required modules: (Note: You do not need to use pip3 or python3 within the virtualenv since it is already a Python3 virtualenv)
    `$ pip install Flask matplotlib numpy pandas altair bokeh holoviews`
18. Create the Flask file. In my case it exists already. => check that the file runs with `$ python grvapp.py` should get a dev server running with output: `* Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)`
    press CTRL+C to quit
19. Setup Apache - need to build for the same version of Python
20. list available apache modeules `$ sudo apt-cache search libapache2*`
21. for Ubuntu, we need to use `$ sudo apt-get install apache2 apache2-dev` (from [mod_wsgi docs](http://bit.ly/2NKJ11N) and [apache docs](http://bit.ly/2NHZ0Oc))
22. From [mod_wsgi docs](http://bit.ly/2NKJ11N), Python must be 3.3 or later, we have 3.6.6 so *should be ok*.
23. Source code for mod_wsgi is [here](http://bit.ly/2NEPNX1), download the latests version and unpack it with `$ tar xvfz mod_wsgi-X.Y.tar.gz` replacing `X.Y` with version number. So for me: `$ tar xvfz mod_wsgi-4.6.4.tar.gz`
24. To setup the package ready for building run the “configure” script from within the source code directory:
    `./configure`
    Then build the source code with 
    `make`
    Then install the Apache module into the standard location for Apache modules as dictated by Apache for your installation, run:
    `make install` or `sudo make install` if permission denied
25. create a wsgi file with
    `$ cd ..` to come back into the correct folder and check paths:
    App path='/home/ubuntu/grv-app-deploy'
    virtualenv path='/home/ubuntu/grv-app-deploy/py36'
    html path='/var/www/html/grv-app-deploy'
    `$ vim app.wsgi`
26. Paste this code into the wsgi file:
    ```python
    activate_this = '/home/ubuntu/grv-app-deploy/py36/bin/activate_this.py'
    with open(activate_this) as f:
	    exec(f.read(), dict(__file__=activate_this))

    import sys
    import logging

    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/html/grv-app-deploy/")

    from grvapp import app as application
    ```
25. Create a symlink so that the project directory appears in /var/www/html
    `$ sudo ln -sT ~/grv-app-deploy /var/www/html/grv-app-deploy`
26. Enable wsgi.
    `$ sudo a2enmod wsgi`
27. Configure apache (you will need to sudo to edit the file)
    `$ sudo vim /etc/apache2/sites-enabled/000-default.conf`
28. Paste this in right after the line with DocumentRoot /var/www/html
    ```python
    WSGIDaemonProcess grv-app-deploy threads=5 socket-timeout=20 memory-limit=850000000 virtual-memory-limit=850000000
    WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi

    <Directory grv-app-deploy>
        WSGIProcessGroup grv-app-deploy
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
    ```
29. Restart the Server:
    `$ sudo apachectl restart`
30. check the IPv4 Public IP at e.g. `54.84.76.221`
    => returns `Internal Server Error`
31. check the logs at: `vim /var/log/apache2/error.log`

    => [wsgi:error] [pid 5719:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [apparently this can be ignored](http://bit.ly/2LIOSUP)

    Stack trace:

    ```
    [Wed Jul 25 07:11:34.348826 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00494: SIGHUP received.  Attempting to restart
    [Wed Jul 25 07:11:34.401663 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
    [Wed Jul 25 07:11:34.401681 2018] [core:notice] [pid 1283:tid 139903880587136] AH00094: Command line: '/usr/sbin/apache2'
    [Wed Jul 25 07:11:38.038388 2018] [wsgi:error] [pid 5719:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [Wed Jul 25 07:11:38.038419 2018] [wsgi:error] [pid 5719:tid 139903778035456]   return f(*args, **kwds)
    [Wed Jul 25 07:11:38.606178 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834] ERROR:flask.app:Exception on / [GET]
    [Wed Jul 25 07:11:38.606198 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834] Traceback (most recent call last):
    [Wed Jul 25 07:11:38.606203 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 2292, in wsgi_app
    [Wed Jul 25 07:11:38.606208 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     response = self.full_dispatch_request()
    [Wed Jul 25 07:11:38.606211 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1815, in full_dispatch_request
    [Wed Jul 25 07:11:38.606213 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     rv = self.handle_user_exception(e)
    [Wed Jul 25 07:11:38.606216 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1718, in handle_user_exception
    [Wed Jul 25 07:11:38.606218 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     reraise(exc_type, exc_value, tb)
    [Wed Jul 25 07:11:38.606221 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/_compat.py", line 35, in reraise
    [Wed Jul 25 07:11:38.606223 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     raise value
    [Wed Jul 25 07:11:38.606226 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1813, in full_dispatch_request
    [Wed Jul 25 07:11:38.606228 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     rv = self.dispatch_request()
    [Wed Jul 25 07:11:38.606230 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1799, in dispatch_request
    [Wed Jul 25 07:11:38.606234 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     return self.view_functions[rule.endpoint](**req.view_args)
    [Wed Jul 25 07:11:38.606236 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/var/www/html/grv-app-deploy/grvapp.py", line 22, in home
    [Wed Jul 25 07:11:38.606238 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     vol = pd.read_csv(volume_file, delim_whitespace=True)
    [Wed Jul 25 07:11:38.606240 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 678, in parser_f
    [Wed Jul 25 07:11:38.606243 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     return _read(filepath_or_buffer, kwds)
    [Wed Jul 25 07:11:38.606245 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 440, in _read
    [Wed Jul 25 07:11:38.606255 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     parser = TextFileReader(filepath_or_buffer, **kwds)
    [Wed Jul 25 07:11:38.606258 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 787, in __init__
    [Wed Jul 25 07:11:38.606260 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     self._make_engine(self.engine)
    [Wed Jul 25 07:11:38.606262 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 1014, in _make_engine
    [Wed Jul 25 07:11:38.606264 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     self._engine = CParserWrapper(self.f, **self.options)
    [Wed Jul 25 07:11:38.606266 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/pandas/io/parsers.py", line 1708, in __init__
    [Wed Jul 25 07:11:38.606269 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]     self._reader = parsers.TextReader(src, **kwds)
    [Wed Jul 25 07:11:38.606272 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "pandas/_libs/parsers.pyx", line 384, in pandas._libs.parsers.TextReader.__cinit__
    [Wed Jul 25 07:11:38.606274 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]   File "pandas/_libs/parsers.pyx", line 695, in pandas._libs.parsers.TextReader._setup_parser_source
    [Wed Jul 25 07:11:38.606278 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834] FileNotFoundError: File b'static/Volumes' does not exist
    [Wed Jul 25 07:11:38.606284 2018] [wsgi:error] [pid 5719:tid 139903778035456] [client 185.50.221.158:44834]
    ```
32. to Fix this error, ensure the absolute path on the server is used: `/var/www/html/grv-app-deploy/static`
33. New error on `/altair` and `/entropy`:

    Stack trace:

    ```
    [Wed Jul 25 09:03:44.155676 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00494: SIGHUP received.  Attempting to restart
    [Wed Jul 25 09:03:44.208533 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
    [Wed Jul 25 09:03:44.208555 2018] [core:notice] [pid 1283:tid 139903880587136] AH00094: Command line: '/usr/sbin/apache2'
    [Wed Jul 25 09:03:52.476335 2018] [wsgi:error] [pid 8110:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [Wed Jul 25 09:03:52.476368 2018] [wsgi:error] [pid 8110:tid 139903778035456]   return f(*args, **kwds)
    [Wed Jul 25 09:03:54.629228 2018] [wsgi:error] [pid 8109:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
    [Wed Jul 25 09:03:54.629260 2018] [wsgi:error] [pid 8109:tid 139903778035456]   return f(*args, **kwds)
    [Wed Jul 25 09:03:55.211838 2018] [wsgi:error] [pid 8109:tid 139903778035456] [client 185.50.221.158:52480] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.334194 2018] [wsgi:error] [pid 8109:tid 139903635539712] [client 185.50.221.158:52480] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.457276 2018] [wsgi:error] [pid 8110:tid 139903769642752] [client 185.50.221.158:52483] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.578340 2018] [wsgi:error] [pid 8110:tid 139903761250048] [client 185.50.221.158:52483] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:55.697555 2018] [wsgi:error] [pid 8110:tid 139903752857344] [client 185.50.221.158:52483] slider_value 0, referer: http://54.84.76.221/
    [Wed Jul 25 09:03:56.929696 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] ERROR:flask.app:Exception on /altair [GET]
    [Wed Jul 25 09:03:56.929727 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] Traceback (most recent call last):
    [Wed Jul 25 09:03:56.929731 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/numpy/lib/npyio.py", line 440, in load
    [Wed Jul 25 09:03:56.929734 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     return pickle.load(fid, **pickle_kwargs)
    [Wed Jul 25 09:03:56.929737 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] _pickle.UnpicklingError: invalid load key, 'v'.
    [Wed Jul 25 09:03:56.929740 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]
    [Wed Jul 25 09:03:56.929743 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] During handling of the above exception, another exception occurred:
    [Wed Jul 25 09:03:56.929745 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]
    [Wed Jul 25 09:03:56.929759 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] Traceback (most recent call last):
    [Wed Jul 25 09:03:56.929762 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 2292, in wsgi_app
    [Wed Jul 25 09:03:56.929765 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     response = self.full_dispatch_request()
    [Wed Jul 25 09:03:56.929767 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1815, in full_dispatch_request
    [Wed Jul 25 09:03:56.929770 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     rv = self.handle_user_exception(e)
    [Wed Jul 25 09:03:56.929772 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1718, in handle_user_exception
    [Wed Jul 25 09:03:56.929783 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     reraise(exc_type, exc_value, tb)
    [Wed Jul 25 09:03:56.929786 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/_compat.py", line 35, in reraise
    [Wed Jul 25 09:03:56.929795 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     raise value
    [Wed Jul 25 09:03:56.929798 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1813, in full_dispatch_request
    [Wed Jul 25 09:03:56.929800 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     rv = self.dispatch_request()
    [Wed Jul 25 09:03:56.929803 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/flask/app.py", line 1799, in dispatch_request
    [Wed Jul 25 09:03:56.929805 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     return self.view_functions[rule.endpoint](**req.view_args)
    [Wed Jul 25 09:03:56.929807 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/var/www/html/grv-app-deploy/grvapp.py", line 109, in altair
    [Wed Jul 25 09:03:56.929810 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     mid_unit = np.load('/var/www/html/grv-app-deploy/static/mid_unit.npy')
    [Wed Jul 25 09:03:56.929812 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]   File "/home/ubuntu/grv-app-deploy/py36/lib/python3.6/site-packages/numpy/lib/npyio.py", line 443, in load
    [Wed Jul 25 09:03:56.929815 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]     "Failed to interpret file %s as a pickle" % repr(file))
    [Wed Jul 25 09:03:56.929819 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483] OSError: Failed to interpret file '/var/www/html/grv-app-deploy/static/mid_unit.npy' as a pickle
    [Wed Jul 25 09:03:56.929825 2018] [wsgi:error] [pid 8110:tid 139903744464640] [client 185.50.221.158:52483]
    ```
34. it appears the large files are not saved on the ubuntu server, if this is the case it would explain the error above, so attempt to use [Git LFS](https://git-lfs.github.com/) as it was used to upload them to github to try to download them:
    follow [this path](https://github.com/git-lfs/git-lfs/wiki/Installation)
    `$ curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash`
    then:
    `$ sudo apt-get install git-lfs`
    and:
    `$ git lfs install`
35. then `$ git lfs pull` to download large files.
36. This worked, but now the page "didn't send any data", perhaps due to large file size
37. use `$ sudo bash -c 'echo > /var/log/apache2/error.log'` to clean error log and `$ sudo apachectl restart` to restart the server
38. While the cause of the 'page didn't send any data' is not clear, Diego Castañeda suggested [mounting the S3 bucket onto the EC2 instance](http://bit.ly/2NIVgMl)
39. Procedure followed to set up an [S3 bucket](https://amzn.to/2LTKDGd)
    - this was initially done directly with the numpy pickles, but as this still does not work...
    - next step is to try with a small image to check the S3 connection is working, 
    - then a large image, to check the file size is not an issue
40. Checking with a small image:
    - upload small image to S3 - done, ok
    - check data available on EC2-mounted S3 bucket - done, ok
    - create new route in app - done, ok
    - test1: page loads with alt-text working correctly. no image but `Distribution` still `In Progress`
    - test2: now `Distribution` is `Available`, reload page: working, image is seen from the `Distribution`.
41. Added a different small image and reloaded, it worked, confirming that S3 bucket, distribution, and mounting all work.
42. Check with a large image (tiff does not display in chrome, trying with a large *.png image):
    - upload large image to S3 - done, ok
    - check data available on EC2-mounted S3 bucket -done, ok
    - create new route in app - done, ok
    - added `WSGIDaemonProcess grv-app-deploy threads=5 socket-timeout=20` (`socket-timeout=20`) to apache2 config file 
    - resized image in template as was showing at default size
    - test: image **loads** *very slowly*
    - interestingly, it loads *a lot faster* from the [direct link](http://d1fmtfaf606jtr.cloudfront.net/nasa_large.png)
    - this means that:
        - S3 **works**
        - mounting of S3 onto EC2 **works**
        - Distribution of data **works**
        - reading data (*.png) through apache2 and mod_wsgi **works**


## Change ownership and permissions on bucket

Run:
- `$ sudo chown -R www-data:www-data /var/www/html/grv-app-deploy/s3bucket/`
- `$ sudo chmod -R 775 /var/www/html/grv-app-deploy/s3bucket/`
- `$ sudo chmod +755 filename`

With all the steps above completed, I attempted again with the /entropy and /altair routes, but I get the following stack trace:

```
[Fri Jul 27 11:35:51.947641 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00494: SIGHUP received.  Attempting to restart
[Fri Jul 27 11:35:52.000537 2018] [mpm_event:notice] [pid 1283:tid 139903880587136] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
[Fri Jul 27 11:35:52.000573 2018] [core:notice] [pid 1283:tid 139903880587136] AH00094: Command line: '/usr/sbin/apache2'
[Fri Jul 27 11:35:56.156415 2018] [wsgi:error] [pid 14021:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:35:56.156444 2018] [wsgi:error] [pid 14021:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:35:58.303229 2018] [wsgi:error] [pid 14022:tid 139903534827264] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:35:58.303260 2018] [wsgi:error] [pid 14022:tid 139903534827264]   return f(*args, **kwds)
[Fri Jul 27 11:36:16.026811 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14021 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:16.026873 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14022 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:17.643632 2018] [wsgi:error] [pid 14101:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:17.643663 2018] [wsgi:error] [pid 14101:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:36:19.031087 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14101 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:24.308217 2018] [wsgi:error] [pid 14132:tid 139903778035456] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:24.308250 2018] [wsgi:error] [pid 14132:tid 139903778035456]   return f(*args, **kwds)
[Fri Jul 27 11:36:26.038966 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14132 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:28.614589 2018] [wsgi:error] [pid 14133:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:28.614618 2018] [wsgi:error] [pid 14133:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:36:30.043712 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14133 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:40.954486 2018] [wsgi:error] [pid 14201:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:40.954517 2018] [wsgi:error] [pid 14201:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:36:48.477437 2018] [wsgi:error] [pid 14242:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:48.477467 2018] [wsgi:error] [pid 14242:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:36:50.064558 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14201 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:50.064629 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14242 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:51.683683 2018] [wsgi:error] [pid 14283:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:51.683726 2018] [wsgi:error] [pid 14283:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:36:53.068704 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14283 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:36:53.623029 2018] [wsgi:error] [pid 14314:tid 139903601968896] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:36:53.623059 2018] [wsgi:error] [pid 14314:tid 139903601968896]   return f(*args, **kwds)
[Fri Jul 27 11:36:55.070960 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14314 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Fri Jul 27 11:37:00.288799 2018] [wsgi:error] [pid 14315:tid 139903744464640] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Fri Jul 27 11:37:00.288830 2018] [wsgi:error] [pid 14315:tid 139903744464640]   return f(*args, **kwds)
[Fri Jul 27 11:37:02.079113 2018] [core:notice] [pid 1283:tid 139903880587136] AH00051: child pid 14315 exit signal Segmentation fault (11), possible coredump in /etc/apache2
```

## Issue with mod_wsgi and pickles

There seems to be an issue with `mod_wsgi` and `numpy.pickles` as shown [here](http://bit.ly/2LHN7dP):

“In practice, what this means is that neither function objects, class objects or instances of classes which are defined in a WSGI application script file should be stored using the “pickle” module.”

So next step:
- flatten numpy.pickles from 3D and 4D to 2D - done
- load to S3 - done
- import to app - done
- change grvapp.py to take in these new arrays - done
- test - done, still no success, stack trace below:

```
[Mon Jul 30 13:30:45.724471 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00494: SIGHUP received.  Attempting to restart
[Mon Jul 30 13:30:45.777475 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
[Mon Jul 30 13:30:45.777496 2018] [core:notice] [pid 14686:tid 140448154359680] AH00094: Command line: '/usr/sbin/apache2'
[Mon Jul 30 13:30:49.387235 2018] [wsgi:error] [pid 15753:tid 140447796074240] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:30:49.387268 2018] [wsgi:error] [pid 15753:tid 140447796074240]   return f(*args, **kwds)
[Mon Jul 30 13:30:58.683843 2018] [wsgi:error] [pid 15754:tid 140447779288832] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:30:58.683875 2018] [wsgi:error] [pid 15754:tid 140447779288832]   return f(*args, **kwds)
[Mon Jul 30 13:30:58.791842 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15753 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Mon Jul 30 13:30:59.793314 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15754 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Mon Jul 30 13:31:00.418507 2018] [wsgi:error] [pid 15832:tid 140447930357504] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:31:00.418538 2018] [wsgi:error] [pid 15832:tid 140447930357504]   return f(*args, **kwds)
[Mon Jul 30 13:31:01.795905 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15832 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Mon Jul 30 13:31:02.029457 2018] [wsgi:error] [pid 15863:tid 140447863215872] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:31:02.029489 2018] [wsgi:error] [pid 15863:tid 140447863215872]   return f(*args, **kwds)
[Mon Jul 30 13:31:02.797090 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15863 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Mon Jul 30 13:31:03.700260 2018] [wsgi:error] [pid 15864:tid 140447880001280] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:31:03.700292 2018] [wsgi:error] [pid 15864:tid 140447880001280]   return f(*args, **kwds)
[Mon Jul 30 13:31:04.799660 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15864 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Mon Jul 30 13:31:10.374125 2018] [wsgi:error] [pid 15935:tid 140447938750208] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:31:10.374157 2018] [wsgi:error] [pid 15935:tid 140447938750208]   return f(*args, **kwds)
[Mon Jul 30 13:31:11.807532 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15935 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Mon Jul 30 13:31:42.011152 2018] [wsgi:error] [pid 15969:tid 140447938750208] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Mon Jul 30 13:31:42.011185 2018] [wsgi:error] [pid 15969:tid 140447938750208]   return f(*args, **kwds)
[Mon Jul 30 13:31:42.841500 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 15969 exit signal Segmentation fault (11), possible coredump in /etc/apache2
```
  
## Changes to WSGIDaemonProcess
Based on [video from mod_wsgi creator Graham Dumpleton](https://youtu.be/H6Q3l11fjU0) and [docs](http://bit.ly/2vq9XfS), apache config file `/etc/apache2/sites-enabled/000-default.conf` to be modified as follows:
 - Edit file with: `$ sudo vim /etc/apache2/sites-enabled/000-default.conf`
 - Paste this in right after the line with DocumentRoot /var/www/html:
 ```python
#WSGIDaemonProcess grv-app-deploy threads=5 socket-timeout=20 memory-limit=850000000 virtual-memory-limit=850000000
WSGIDaemonProcess grv-app-deploy display-name='%{GROUP}' lang='en_US.UTF-8' locale='en_US.UTF-8' threads=5 queue-timeout=45 \
    socket-timeout=60 connect-timeout=15 request-timeout=60 inactivity-timeout=0 startup-timeout=15 deadlock-timeout=60 \
    graceful-timeout=15 eviction-timeout=0 restart-interval=0 shutdown-timeout=5 maximum-requests=0 \
    memory-limit=850000000 virtual-memory-limit=850000000

WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi

<Directory grv-app-deploy>
    WSGIProcessGroup grv-app-deploy
    WSGIApplicationGroup %{GLOBAL}
    Order deny,allow
    Allow from all
</Directory>
 ```
=> change made, no change to errors, error stack trace below:

```
[Wed Aug 01 14:55:13.347443 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00494: SIGHUP received.  Attempting to restart
[Wed Aug 01 14:55:13.400283 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
[Wed Aug 01 14:55:13.400306 2018] [core:notice] [pid 14686:tid 140448154359680] AH00094: Command line: '/usr/sbin/apache2'
[Wed Aug 01 14:55:18.653625 2018] [wsgi:error] [pid 27081:tid 140447779288832] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Wed Aug 01 14:55:18.653658 2018] [wsgi:error] [pid 27081:tid 140447779288832]   return f(*args, **kwds)
[Wed Aug 01 14:55:19.406750 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 27081 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Wed Aug 01 14:55:20.232231 2018] [wsgi:error] [pid 27082:tid 140447980713728] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Wed Aug 01 14:55:20.232262 2018] [wsgi:error] [pid 27082:tid 140447980713728]   return f(*args, **kwds)
[Wed Aug 01 14:55:21.409273 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 27082 exit signal Segmentation fault (11), possible coredump in /etc/apache2
[Wed Aug 01 14:55:21.829169 2018] [wsgi:error] [pid 27155:tid 140447938750208] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Wed Aug 01 14:55:21.829201 2018] [wsgi:error] [pid 27155:tid 140447938750208]   return f(*args, **kwds)
[Wed Aug 01 14:55:23.411768 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 27155 exit signal Segmentation fault (11), possible coredump in /etc/apache2
```

## Changes to WSGIDaemonProcess

Based on same as above, added following line to code: `WSGIRestrictEmbedded On`
 - this **must** be added **above** the `<VirtualHost *:80>` line.

=> The server does not run, stack trace:

```
[Thu Aug 02 06:40:50.190275 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
[Thu Aug 02 06:40:50.190299 2018] [core:notice] [pid 14686:tid 140448154359680] AH00094: Command line: '/usr/sbin/apache2'
[Thu Aug 02 06:42:13.784533 2018] [wsgi:error] [pid 4014:tid 140447880001280] [client 185.50.221.158:58598] Embedded mode of mod_wsgi disabled by runtime configuration: /var/www/html/grv-app-deploy/app.wsgi
```

The fact that `Embedded mode of mod_wsgi disabled by runtime configuration` suggests the `WSGIRestrictEmbedded On` was placed correctly _but_ that a process _is trying_ to run in embedded mode, which mod_wsgi strongly dissaproves.
=> need to find out how to fix this.

## Changes to WSGIProcessGroup

Based on same as above (docs [here](bit.ly/2O3jZeE)), modified following line as follows:

Line before: `WSGIProcessGroup grv-app-deploy`
Line after: `WSGIProcessGroup %{GLOBAL}`

=> error persisted:

```
[Thu Aug 02 06:54:55.062531 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00494: SIGHUP received.  Attempting to restart
[Thu Aug 02 06:54:55.115037 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
[Thu Aug 02 06:54:55.115055 2018] [core:notice] [pid 14686:tid 140448154359680] AH00094: Command line: '/usr/sbin/apache2'
[Thu Aug 02 06:55:05.159824 2018] [wsgi:error] [pid 4129:tid 140447980713728] [client 185.50.221.158:58695] Embedded mode of mod_wsgi disabled by runtime configuration: /var/www/html/grv-app-deploy/app.wsgi
```

`Embedded mode of mod_wsgi disabled by runtime configuration` still being returned. 

=> Code broken. Something running in embedded mode?

`/etc/apache2/sites-enabled/000-default.conf` current state:

```Python
WSGIRestrictEmbedded On

<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        WSGIDaemonProcess grv-app-deploy display-name='%{GROUP}' lang='en_US.UTF-8' locale='en_US.UTF-8' threads=5 queue-timeout=45 \
                socket-timeout=60 connect-timeout=15 request-timeout=60 inactivity-timeout=0 startup-timeout=15 deadlock-timeout=60 \
                graceful-timeout=15 eviction-timeout=0 restart-interval=0 shutdown-timeout=5 maximum-requests=0 \
                memory-limit=850000000 virtual-memory-limit=850000000

        WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi

        <Directory grv-app-deploy>
                #WSGIProcessGroup grv-app-deploy
                WSGIProcessGroup %{GLOBAL}
                WSGIApplicationGroup %{GLOBAL}
                Order deny,allow
                Allow from all
        </Directory>

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

From [Checking your installation](bit.ly/2O3YGtf), switch back `WSGIProcessGroup` to previous value:
 - `WSGIProcessGroup grv-app-deploy`

updates to config file:
 - `WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi` changed to `WSGIScriptAlias /grv-app-deploy /var/www/html/grv-app-deploy/app.wsgi`
 - `WSGIProcessGroup grv-app-deploy` moved out of `<Directory>`:

```Python
WSGIRestrictEmbedded On

<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        WSGIDaemonProcess grv-app-deploy display-name='%{GROUP}' lang='en_US.UTF-8' locale='en_US.UTF-8' threads=5 queue-timeout=45 \
                socket-timeout=60 connect-timeout=15 request-timeout=60 inactivity-timeout=0 startup-timeout=15 deadlock-timeout=60 \
                graceful-timeout=15 eviction-timeout=0 restart-interval=0 shutdown-timeout=5 maximum-requests=0 \
                memory-limit=850000000 virtual-memory-limit=850000000

        WSGIProcessGroup grv-app-deploy

        WSGIScriptAlias / /var/www/html/grv-app-deploy/app.wsgi

        <Directory grv-app-deploy>
                #WSGIProcessGroup grv-app-deploy
                WSGIApplicationGroup %{GLOBAL}
                Order deny,allow
                Allow from all
        </Directory>
        ...
```

=> Error log:

```
[Thu Aug 02 07:23:42.914963 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00494: SIGHUP received.  Attempting to restart
[Thu Aug 02 07:23:42.967653 2018] [mpm_event:notice] [pid 14686:tid 140448154359680] AH00489: Apache/2.4.18 (Ubuntu) mod_wsgi/4.6.4 Python/3.6 configured -- resuming normal operations
[Thu Aug 02 07:23:42.967675 2018] [core:notice] [pid 14686:tid 140448154359680] AH00094: Command line: '/usr/sbin/apache2'
[Thu Aug 02 07:23:57.236958 2018] [wsgi:error] [pid 4688:tid 140448008148736] /usr/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: numpy.dtype size changed, may indicate binary incompatibility. Expected 96, got 88
[Thu Aug 02 07:23:57.236991 2018] [wsgi:error] [pid 4688:tid 140448008148736]   return f(*args, **kwds)
[Thu Aug 02 07:23:57.949125 2018] [wsgi:error] [pid 4690:tid 140447880001280] [client 185.50.221.158:58811] Truncated or oversized response headers received from daemon process 'grv-app-deploy': /var/www/html/grv-app-deploy/app.wsgi
[Thu Aug 02 07:23:57.983496 2018] [core:notice] [pid 14686:tid 140448154359680] AH00051: child pid 4688 exit signal Segmentation fault (11), possible coredump in /etc/apache2
```
=> `Truncated or oversized response headers received from daemon process 'grv-app-deploy'`
Also now with `WSGIRestrictEmbedded On` set, the app still runs on first page, suggesting it is _no longer_ running in embedded mode.

[This](http://bit.ly/2OAsdfj) github issue suggested setting `LogLevel info`, this yielded the following information:
```
[Thu Aug 02 07:40:15.473822 2018] [wsgi:info] [pid 4961:tid 140448154359680] mod_wsgi (pid=4961): Attach interpreter ''.
[Thu Aug 02 07:50:23.733644 2018] [wsgi:info] [pid 4961:tid 140448008173312] mod_wsgi (pid=4961): Create interpreter 'ip-172-31-29-84.ec2.internal|'.
```

`wsgi:info] [pid 4961:tid 140448154359680] mod_wsgi (pid=4961): Attach interpreter ''` pointed to [this post](http://bit.ly/2ODhMYs) which suggests that there is a problem with the Python installation and that `Python installation needs to have been installed with the --enable-shared option given to its 'configure' command.`. More information is available [here](http://bit.ly/2vcoahb). This suggests I should _reinstall_ `python` and therefore _rebuild_ `mod_wsgi` as it is built against a specific Python version.

**However**: I do not know how to clean old versions completely so perhaps I should just rebuild the whole app... not ideal.
