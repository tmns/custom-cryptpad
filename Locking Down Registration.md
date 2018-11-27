# Locking Down Registration

The basic flow goes like this:
1. User inputs a registration key via the registration form (`/register/`). 
2. This key is hashed and compared via `bcryptjs` with a hash defined in `credential.js`.
3. If the hashes match, user registration is succeessful. 
4. Else, user is shown an Invalid Registration Key error.

Now let's start!

* Begin by grabbing dcodeIO's `bcryptjs` and throwing it in `data/customize/src` (the location you put it isn't terribly important, however):

```
$ wget https://raw.githubusercontent.com/dcodeIO/bcrypt.js/master/dist/bcrypt.min.js 
$ mv bcrypt.min.js data/customize/src/
```

*Alternatively, you can add `"bcryptjs": "2.4.3"` to the project's `bower.json` and rebuild the project. Note that if you go this route, the location of `bcrypt.min.js` will be different (most likely located in `bower-components/`) than in this example. *

* Choose a value for the registration key (or generate one randomly). Just make sure the value does not contain slashes, as it currently causes things to fail.
* Now you need to derive the hash for the value you've chosen, this can be achieved by doing something like this:

```
$ sudo npm install bcryptjs
[...]
$ node
> var bcrypt = require('bcryptjs');
> bcrypt.hashSync('reg_key_value', 10);
'hash_of_reg_key_value'
```
*Remember to store `reg_key_value` (ie the plaintext registration key) somewhere secure, such as a password manager. When you want someone to register you will need to transfer this value to them via a secure channel.*

* Now, we must edit the actual project files to add our registration key validation logic. 
* First, let's add our registration key hash value to `data/customize/credential.js` and create a simple function that validates the user submitted value (from the `/register/` form) exists and is of `type string`. Put the following where appropriate in `credential.js`:

```
[...]
Cred.REGKEY_HASH = 'hash_of_reg_key_value';
[...]
Cred.isValidRegkey = function (regkey) {
	return !!(regkey && isString(regkey));
};
[...]
```
* Next, we make sure to include the bcrypt package in `data/customize/login.js`, write up the function that checks if the user submitted value hashes to our predefined hash (using bcrypt's `compareSync(value, hash)`, and return an error message to the user if it doesn't:

*Note: I will be including some original code throughout these examples (ie code that doesn't directly relate to registration key logic) for context.*

```
define([
    'jquery',
    '/bower_components/chainpad-listmap/chainpad-listmap.js',
    '/bower_components/chainpad-crypto/crypto.js',
    '/common/common-util.js',
    '/common/outer/network-config.js',
    '/customize/credential.js',
    '/bower_components/chainpad/chainpad.dist.js',
    '/common/common-realtime.js',
    '/common/common-constants.js',
    '/common/common-interface.js',
    '/common/common-feedback.js',
    '/common/outer/local-store.js',
    '/customize/messages.js',
    '/bower_components/nthen/index.js',
    '/common/outer/login-block.js',
    '/common/common-hash.js',
    '/customize/src/bcrypt.min.js',

    '/bower_components/tweetnacl/nacl-fast.min.js',
    '/bower_components/scrypt-async/scrypt-async.min.js', // better load speed
], function ($, Listmap, Crypto, Util, NetConfig, Cred, ChainPad, Realtime, Constants, UI,
            Feedback, LocalStore, Messages, nThen, Block, Hash, Bcrypt) {

[...]
	//Exports.loginOrRegister = function (uname, passwd, isRegister, shouldImport, cb) {
	Exports.loginOrRegister = function (uname, passwd, regkey, isRegister, shouldImport, cb) {
[...]
	// If registering, check if submitted regkey is valid && matches the defined value
	if (isRegister) {
		if (!Cred.isValidRegkey(regkey) || !Bcrypt.compareSync(regkey, Cred.REGKEY_HASH)) {
			return void cb('INVAL_REGKEY');
		}
	}
[...]
	//Exports.loginOrRegisterUI = function (uname, passwd, isRegister, shouldImport, testing, test) {
	Exports.loginOrRegisterUI = function (uname, passwd, regkey, isRegister, shouldImport, testing, test) {
[...]
        window.setTimeout(function () {
            //Exports.loginOrRegister(uname, passwd, regkey, isRegister, shouldImport, function (err, result) {
            Exports.loginOrRegister(uname, passwd, regkey, isRegister, shouldImport, function (err, result) {
[...]
              if (err) {
                switch (err) {
[...]
                  case 'INVAL_REGKEY':
                    UI.removeLoadingScreen(function () {
                      UI.alert(Messages.register_invalRegkey, function () {
                        registering = false;
                      });
                    });
                    break;
[...]
```
* Now that the validation logic is done, we need to make sure it connects with what's presented to the user. Thus, we have to edit the view. 
* First, we define the values for the `/register/` form field's placeholder and the error message presented to the user if the registration key is invalid in `www/common/translations/messages.js` (or whichever translation file you're using):

```
[...]
	out.register_regkey = 'Registration Key';
	out.register_invalRegkey = 'Invalid Registration Key.';
[...]
```

* Now, we can edit the application's view so that the registration form will present the user with a field to input the registration key, which will use the placeholder defined above. To accomplish this, we add a few lines to `data/customize/pages/register.js`:

```
[...]
	h('input.form-control#password', {
		type: 'password',
		placeholder: Msg.login_password,
	}),
	h('input.form-control#password-confirm', {
		type: 'password',
		placeholder: Msg.login_confirm,
	}),
	h('input.form-control#regkey', {
		type: 'password',
		placeholder: Msg.register_regkey,
	}),
[...]
```

* Next, we will need to customize both `www/register/main.js` and `www/login/main.js`, so we create the following directories and copy the respective `main.js` files to them:

```
$ mkdir data/customize/register && cp www/register/main.js data/customize/register
$ mkdir data/customzie/login && cp www/login/main.js data/customize/login
```

* First we make the necessary edits to `data/customize/register/main.js`, which will enable the app to obtain the value entered into the registration form field configured above and make the correct function call:

```
[...]
	var $uname = $('#username');
	var $passwd = $('#password');
	var $confirm = $('#password-confirm');
	var $regkey = $('#regkey');
[...]
	// [ $uname, $passwd, $confirm]
	[ $uname, $passwd, $confirm, $regkey ]
[...]
	var uname = $uname.val();
	var passwd = $passwd.val();
	var confirmPassword = $confirm.val();
	var regkey = $regkey.val();
[...]
	//Login.loginOrRegisterUI(uname, passwd, true, shouldImport, Test.testing, function () {
	Login.loginOrRegisterUI(uname, passwd, regkey, true, shouldImport, Test.testing, function () {
[...]

```
* Now we just need to make a simple edit to `data/customize/login/main.js` so that the correct function call is made:

```
[...]
	var uname = $uname.val();
	var passwd = $passwd.val();
	var regkey = null;
	// Login.loginOrRegisterUI(uname, passwd, false, shouldImport, Test.testing, function () {
	Login.loginOrRegisterUI(uname, passwd, regkey, false, shouldImport, Test.testing, function () {
[...]
```
* And finally, we edit `data/customize/template.js` to serve our customized `main.js` files instead of those in `www/`:

```
[...]
	if (/^\/user\//.test(pathname)) {
		require([ '/user/main.js'], function () {});
	} else if (/^\/register\//.test(pathname)) {
		// require([ '/register/main.js' ], function () {});
		require([ '/customize/register/main.js' ], function () {});
	} else if (/^\/login\//.test(pathname)) {
		// require([ '/login/main.js' ], function () {});
		require([ '/customize/login/main.js' ], function () {});
	} else if (/^\/($|^\/index\.html$)/.test(pathname)) {
[...]
```
* That's it! Now, if you start the container and navigate to  `/register/`, there should be an extra form field with the placeholder `Registration Key` under the confirm password field. Leaving it blank or inputting a value other than the pre-defined value (`reg_key_value`) should then result in an `Invalid Registration Key` error returned to the user.
