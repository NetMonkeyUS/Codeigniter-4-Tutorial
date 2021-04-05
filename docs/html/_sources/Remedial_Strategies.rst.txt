Remedial strategies
===================
Move css and images and js folders
----------------------------------
You can have your assets (js css & images) in the project root on your local machine and in your public folder on your live hosting. This is probably the easiest answer but a little disturbing. You shouldn't have to trick things right out of the box.

.. todo:: Is this really true or did I just screw something up on the first goround? Need to retest.

Turbocharge file structure (Better but more complicated)
--------------------------------------------------------
I forked this from this web page: https://forum.codeigniter.com/thread-75774.html

- If you clone from github your project folder is called 'Codeigniter4'. If you download, it might be called 'framework-4.1.1' or something. You can call it anything but just for clarity rename it 'Codeigniter4' (at least while trying out this tutorial).
- Inside the 'Codeigniter4' folder, change the name of your 'public' folder to 'public_html' to mimic the folder structure of your hosting.
- Make a new folder inside 'public_html' called 'mydomain.com' or whatever locally duplicates your live domain. (Always replace the term 'mydomain' with your real domain name.)
- Move EVERYTHING in 'public_html' into the subfolder 'mydomain.com'.
- Create a new folder named 'ci4_code' (or whatever you want to call it) inside the 'Codeigniter4' folder.
- Move everything that ISN'T in the 'public_html' folder into the 'ci4_code' folder EXCEPT the file 'spark', README.md, and LICENSE (and the github files and folders if you cloned, i.e. .git, .github, .editorconfig, .nojekyll, .gitattributes but MOVE .gitignore -- it's supposed to be in the 'ci4_code' folder).
- Change FCPATH in 'public_html/index.php' (at or about line 28) (my comments are in CAPITALS)::

	// Load our paths config file
	// This is the line that might need to be changed, depending on your folder structure.
	//CHANGED THIS PER https://forum.codeigniter.com/thread-75774.html
	require realpath(FCPATH . '../../ci4_code/app/Config/Paths.php') ?: FCPATH . '../../ci4_code/app/Config/Paths.php';
	// ^^^ Change this if you move your application folder

- Change 'FCPATH' and 'realpath' in 'Codeigniter4/spark' file (starting at about line 40)::

	// Path to the front controller
	//CHANGED THIS PER https://forum.codeigniter.com/thread-75774.html
	define('FCPATH', __DIR__ . DIRECTORY_SEPARATOR . 'public_html'. DIRECTORY_SEPARATOR . 'mydomain.com' . DIRECTORY_SEPARATOR);

	// Load our paths config file
	//CHANGED THIS TO POINT TO RIGHT FILE
	require realpath('ci4_code/app/Config/Paths.php') ?: 'ci4_code/app/Config/Paths.php';
	// ^^^ Change this line if you move your application folder

- The 'Codeigniter4/spark' file can have any name. Just make sure your code editor doesn't save an extension like '.txt' or '.php' on the end. We'll call it 'spark_mydomain'. That way if you want to use CodeIgniter4 on any other local domains, all you have to do is change the 'spark' file.

Here's where it gets tricky. With where you are so far, if you have multiple domains using the 'ci4_code' folder they are all going to be using the same '.env' file which I don't think will work. There might be other ways of getting around this, but what follows is the work-around I used. It seems to work, but I don't know about security -- and as well you have to alter a CodeIgniter system file which is supposed to be a 'no-no'.

- The first thing I tried was adding a suffix to the '.env' file like '.env.localhost'. This worked great until you tried to use the command 'php spark_mydomain migrate'. It turns out spark wants the '.env' file to be '.env' and nothing else. So my solution was to put each '.env' file in their own directory, so the path to the '.env' file would be thus: 'ci4_code/mydomain.com/.env'. Here's the code for the DotEnv.php file::

	public function __construct(string $path, string $file = '.env')
	{
		//FCPATH IS 'Front controller path' per https://stackoverflow.com/questions/13992074/codeigniter-path-constants-definitions
		//GET 'mydomain.com' from FCPATH
		$envdir = basename(FCPATH);
		//ADD $envdir (ENVELOPE DIRECTORY) TO PATH
		$this->path = rtrim($path, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . $envdir . DIRECTORY_SEPARATOR . $file;
		
	}

- This is okay with localhost but we need something more better for 'live'::

	public function __construct(string $path, string $file = '.env')
	{
		/* SOURCES 
		https://stackoverflow.com/questions/17201170/php-how-to-get-the-base-domain-url
		https://expressionengine.com/blog/http-host-and-server-name-security-issues */

		if ($_SERVER['SERVER_NAME'] === 'localhost'){
			$domain = 'localhost';
		} else {
			$domain = basename(FCPATH);
		}
		// JUST FOR TESTING
		echo "Serving from domain: " . $domain;
				
		//BACK TO ORIGINAL CODE except added 'domain' folder to hold '.env' file
		$this->path = rtrim($path, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . $domain . DIRECTORY_SEPARATOR . $file;
		
	}

- THIS STUFF DOESN'T WORK BECAUSE spark migrate hangs up on any envelope file that is not exactly '.env'

- Go to your '.env' file. Save a copy as '.env.mydomain.com' (where 'mydomain.com' is your real domain). ALSO, save a copy as '.env.localhost'. Put your localhost base_URL and database settings in '.env.localhost'.

- Open this file 'system/Config/DotEnv.php'. (Wherever you can find it -- they move it around with different versions).

- At about line 38 in 'DotEnv.php' it tells the app where to find the '.env' file. At first I simply appended '.mydomain.com' automatically to the end of the '$file' variable using $_SERVER['SERVER_NAME'] ::

	public function __construct(string $path, string $file = '.env')
	{
		/* ADDED THIS FROM https://stackoverflow.com/questions/17201170/php-how-to-get-the-base-domain-url */
		
		$file = $file . '.' . ($_SERVER['SERVER_NAME']);
				
		$this->path = rtrim($path, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . $file;
	}
	
- HOWEVER, evidently using $SERVER('SERVER_NAME') is a security risk. To get around this risk it is better to make an array of the 'okay' server names and check against it so I modified the code to as follows::
	
	public function __construct(string $path, string $file = '.env')
	{
		/* SOURCES: 
		https://stackoverflow.com/questions/17201170/php-how-to-get-the-base-domain-url
		https://expressionengine.com/blog/http-host-and-server-name-security-issues 
		*/

		$domains = array('mydomain.com', 'myotherdomain.com', 'localhost');

		if (in_array($_SERVER['SERVER_NAME'], $domains)){
			$domain = $_SERVER['SERVER_NAME'];
			//echo "Serving from verfied domain: " . $domain;	// for testing
			$domain = '.' . $domain; 						
			} else {	
				//echo 'error -- Potential hack attack';		// for testing
				$domain = '';
			} 
		
		/*BACK TO ORIGINAL CODE except added '.domain'*/
		$this->path = rtrim($path, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . $file . $domain;
		
	}
	
- BUT ANOTHER PROBLEM is that in order to modify your server names (in case you add or subtract a domain) you have to dig way into the Codeigniter framework for the DotEnv.php file. To avoid this hassle I cut out the modified code and pasted it into a file called 'ValidDomains.php' which we will save in the root ('ci4_code') of the project. Luckily the '$path' variable is already maps where we want to be so I just prepended it to 'ValidDomains.php'. Here is the end result::

	'ValidDomains.php' in root (aka 'ci4_code/ValidDomains.php')
		
		<?php

			/* SOURCES:
			https://stackoverflow.com/questions/17201170/php-how-to-get-the-base-domain-url
			https://expressionengine.com/blog/http-host-and-server-name-security-issues 
			https://stackoverflow.com/questions/13234122/index-server-name-not-exist */
			
			$server_name = (isset($_SERVER['SERVER_NAME']))?	// check if SERVER_NAME is set
			$_SERVER['SERVER_NAME'] :							// if yes, use HTTP header value
			php_uname("n");										// if no, use php_uname()
			
			$domains = [
				'mydomain.com',
				'myotherdomain.org',
				'localhost'
			];
			
			if (in_array($server_name, $domains)){
				$domain = $server_name;
				//echo "Serving from verfied domain: " . $domain; 	//for testing
				$domain = '.' . $domain;			//add the dot here is better
				
				} else {	
					//echo 'error -- Potential hack attack'; 		//for testing
					$domain = '';
				}
				
			$file = ($file . $domain);
	
	And the altered code within '/vendor/codeigniter4/framework/system/Config/DotEnv.php' (at about line 40)::
		
		public function __construct(string $path, string $file = '.env')
		{
			/* put in higher directory so easier to access */
			require ($path . DIRECTORY_SEPARATOR . 'ValidDomains.php');	
					
			/*BACK TO ORIGINAL CODE*/
			$this->path = rtrim($path, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . $file;
			
		}

- The last step is to upload the modified folder structure to your live domain in the corresponding folders, and change the '.env.mydomain.com' file to match your live info. Your 'Config/App.php' and 'Config/Database' don't seem to have any effect as long as you have a working dotenv file (in our case '.env.mydomain.com'). 

Also you should change from 'development' to 'production' in your '.env.mydomain.com' file on your live server. You might want to add your custom dotenv file (.env.mydomain.com) to your '.gitignore' file on the same level though I don't know enough about it to know for sure. Like so::

   #-----------------------
   ADDED THIS for my multi-domain work-around
   #-----------------------
   .env*


With this set up you should be good to develop locally and just drag and drop to your live site.

Pros and cons to this structure
-------------------------------

The plusses of this structure is that it is more secure. Most of your code is not web accessible. Also, it takes less space on your server if you are going to run multiple websites.


The 'cons' of this way of doing it this way is that your 'views' folder and 'Config/Routes.php' are going to be more crowded, with multiple websites being referenced. There are ways to adapt to this like having separate subfolders for each website in the 'views' folder, and grouping the different website routes together somehow. 


But there are advantages even in the crowdedness. You might be able to reuse controllers on different sites. Maybe your 'Login' controller or some other controller will work just fine on both 'mydomain.com' and 'my_otherdomain.com' even though it accesses a different database. Your unique content will be in your 'public_html' folder. Your structure is in the 'app' folder and might not need to be unique.





.. todo:: test with real install
