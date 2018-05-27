---
layout: post
title:  "Hype development, suspicious engineering"
author: David Gross
date:   2018-05-26 17:21
---

After a few years of loyalty to QtCreator as C++ IDE, I decided to give a try to [Visual Studio Code](https://code.visualstudio.com/). Very positively surprised
by its simplicity of usage, its built-in features and the quality of extensions, I thought about going one step further and write an extension.

In the [Hello World extension example](https://code.visualstudio.com/docs/extensions/example-hello-world), it is adviced to generate the extension skeleton
through a Node.js generator:

>
> We have written a Yeoman generator to help get you started.
> `npm install -g yo generator-code`
> `yo code`
>

The thing is, running this command took almost as long as the installation of my entire Linux distribution.

For this small code generator, I had to install **1066** dependencies. So what kind of dependencies are we having here? Let's have a look. Here are a few of them:

  * *isobject*, six times
  * *kindof*, twenty two times
  * *is-obj*, once
  * *unique-string*, once
  * *path-exists*, four times
  * *is-number*, four times
  * *isarray*, four times
  * *number-is-nan*, once

Curious about the code behind those, I glanced a their github. This is `path-exists`:

```js
module.exports.sync = fp => {
	try {
		fs.accessSync(fp);
		return true;
	} catch (err) {
		return false;
	}
};
```
<br />

So this module is really only converting an exception to a boolean. Let's look at this is `number-is-nan`:
```js
module.exports = function (x) {
	return x !== x;
};
```
<br />

Alright, I guess this implementation "makes sense"... this is the `unique-string` module:
```js
module.exports = () => cryptoRandomString(32);
```
<br />

... its dependency `crypto-random-string`:
```js
module.exports = len => {
	if (!Number.isFinite(len)) {
		throw new TypeError('Expected a finite number');
	}

	return crypto.randomBytes(Math.ceil(len / 2)).toString('hex').slice(0, len);
};
```
<br />

`unique-string` is just a function call to `cryptoRandomString(32)`, and ther three out of four lines code check that the argument is a finite number &mdash;
slightly surprised the developer didn't use the module `is-finite` here by the way!
<br />


Suspicious engineering
----------------------
Is this really needed? What does it bring? Why would I add to my code an additional dependency, which means downloading an external code, import it, and call its 
only one-liner just to check if a number is `NaN`, when I call myself use `!==`, or the ECMAScript 2016 function isNaN if I know my variable is a number?

Same question this `path-exists`. Same about this odd `crypto-random-string` that returns an "MD5"-style string... 

Adding a dependency to a codebase is not free. There should be a strong reason for doing so. For example, I see some good reasons to have one main core library 
around type information. Such library would then group many of the modules like `is-array`, `is-buffer`, `kind-of` and so on, but in a *consistent* way. These modules
depend anyway from each other, so why not group them?

On the top of that, having hundreds of small independent modules can only bring confusion: what is the difference between the packages `is-array`, `isarray`, 
`is-arrayish`, `is-array-like`? I will eventually read their implementation, which slightly defeats the purpose of using a library.

Anyway, I am clearly not saying that something is wrong in `Node` or `npm`, but it seems that the direction some developers took is slightly extreme, and questionable. 

And of course, eventually, after some long minutes, the installation of my Visual Studio Code extension generator failed in the middle of the installation of the dependencies.


```
# npm install --dry-run -g yo generator-code

▄ ╢░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:inquirer → resolv ▄ ╢██████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:yeoman-environmen ▀ ╢██████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:micromatch → requ ▌ ╢██████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:micromatch → 304  ▌ ╢██████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:urix → request    ▀ ╢██████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:yosay → get       ▀ ╢██████████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:foreachasync → re ▐ ╢████████████████████████████████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
loadDep:debug → mapToRegi ▄ ╢████████████████████████████████████████████████████████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░╟
/usr/local/lib
├─┬ generator-code@1.1.31 
│ ├── assert-plus@1.0.0 
│ ├─┬ chalk@2.4.1 
│ │ ├─┬ ansi-styles@3.2.1 
│ │ │ └─┬ color-convert@1.9.1 
│ │ │   └── color-name@1.1.3 
│ │ ├── escape-string-regexp@1.0.5 
│ │ └─┬ supports-color@5.4.0 
│ │   └── has-flag@3.0.0 
│ ├─┬ request@2.87.0 
│ │ ├── aws-sign2@0.7.0 
│ │ ├── aws4@1.7.0 
│ │ ├── caseless@0.12.0 
│ │ ├─┬ combined-stream@1.0.6 
│ │ │ └── delayed-stream@1.0.0 
│ │ ├── extend@3.0.1 
│ │ ├── forever-agent@0.6.1 
│ │ ├─┬ form-data@2.3.2 
│ │ │ └── asynckit@0.4.0 
│ │ ├─┬ har-validator@5.0.3 
│ │ │ ├─┬ ajv@5.5.2 
│ │ │ │ ├── co@4.6.0 
│ │ │ │ ├── fast-deep-equal@1.1.0 
│ │ │ │ ├── fast-json-stable-stringify@2.0.0 
│ │ │ │ └── json-schema-traverse@0.3.1 
│ │ │ └── har-schema@2.0.0 
│ │ ├─┬ http-signature@1.2.0 
│ │ │ ├─┬ jsprim@1.4.1 
│ │ │ │ ├── extsprintf@1.3.0 
│ │ │ │ ├── json-schema@0.2.3 
│ │ │ │ └── verror@1.10.0 
│ │ │ └─┬ sshpk@1.14.1 
│ │ │   ├── asn1@0.2.3 
│ │ │   ├── bcrypt-pbkdf@1.0.1 
│ │ │   ├── dashdash@1.14.1 
│ │ │   ├── ecc-jsbn@0.1.1 
│ │ │   ├── getpass@0.1.7 
│ │ │   ├── jsbn@0.1.1 
│ │ │   └── tweetnacl@0.14.5 
│ │ ├── is-typedarray@1.0.0 
│ │ ├── isstream@0.1.2 
│ │ ├── json-stringify-safe@5.0.1 
│ │ ├─┬ mime-types@2.1.18 
│ │ │ └── mime-db@1.33.0 
│ │ ├── oauth-sign@0.8.2 
│ │ ├── performance-now@2.1.0 
│ │ ├── qs@6.5.2 
│ │ ├── safe-buffer@5.1.2 
│ │ ├─┬ tough-cookie@2.3.4 
│ │ │ └── punycode@1.4.1 
│ │ ├── tunnel-agent@0.6.0 
│ │ └── uuid@3.2.1 
│ ├─┬ sanitize-filename@1.6.1 
│ │ └─┬ truncate-utf8-bytes@1.0.2 
│ │   └── utf8-byte-length@1.0.4 
│ ├── sax@1.2.4 
│ ├─┬ yeoman-generator@0.24.1 
│ │ ├── async@2.6.1 
│ │ ├─┬ chalk@1.1.3 
│ │ │ ├── ansi-styles@2.2.1 
│ │ │ ├── has-ansi@2.0.0 
│ │ │ └── supports-color@2.0.0 
│ │ ├─┬ class-extend@0.1.2 
│ │ │ └── object-assign@2.1.1 
│ │ ├─┬ cli-table@0.3.1 
│ │ │ └── colors@1.0.3 
│ │ ├─┬ cross-spawn@4.0.2 
│ │ │ ├─┬ lru-cache@4.1.3 
│ │ │ │ ├── pseudomap@1.0.2 
│ │ │ │ └── yallist@2.1.2 
│ │ │ └─┬ which@1.3.1 
│ │ │   └── isexe@2.0.0 
│ │ ├─┬ dargs@4.1.0 
│ │ │ └── number-is-nan@1.0.1 
│ │ ├─┬ dateformat@1.0.12 
│ │ │ ├── get-stdin@4.0.1 
│ │ │ └─┬ meow@3.7.0 
│ │ │   ├─┬ camelcase-keys@2.1.0 
│ │ │   │ └── camelcase@2.1.1 
│ │ │   ├── decamelize@1.2.0 
│ │ │   ├─┬ loud-rejection@1.6.0 
│ │ │   │ ├─┬ currently-unhandled@0.4.1 
│ │ │   │ │ └── array-find-index@1.0.2 
│ │ │   │ └── signal-exit@3.0.2 
│ │ │   ├── map-obj@1.0.1 
│ │ │   ├─┬ normalize-package-data@2.4.0 
│ │ │   │ ├── hosted-git-info@2.6.0 
│ │ │   │ ├─┬ is-builtin-module@1.0.0 
│ │ │   │ │ └── builtin-modules@1.1.1 
│ │ │   │ ├── semver@5.5.0 
│ │ │   │ └─┬ validate-npm-package-license@3.0.3 
│ │ │   │   ├─┬ spdx-correct@3.0.0 
│ │ │   │   │ └── spdx-license-ids@3.0.0 
│ │ │   │   └─┬ spdx-expression-parse@3.0.0 
│ │ │   │     └── spdx-exceptions@2.1.0 
│ │ │   ├── object-assign@4.1.1 
│ │ │   ├─┬ redent@1.0.0 
│ │ │   │ ├─┬ indent-string@2.1.0 
│ │ │   │ │ └─┬ repeating@2.0.1 
│ │ │   │ │   └── is-finite@1.0.2 
│ │ │   │ └── strip-indent@1.0.1 
│ │ │   └── trim-newlines@1.0.0 
│ │ ├─┬ debug@2.6.9 
│ │ │ └── ms@2.0.0 
│ │ ├── detect-conflict@1.0.1 
│ │ ├─┬ error@7.0.2 
│ │ │ ├── string-template@0.2.1 
│ │ │ └── xtend@4.0.1 
│ │ ├─┬ find-up@1.1.2 
│ │ │ └─┬ pinkie-promise@2.0.1 
│ │ │   └── pinkie@2.0.4 
│ │ ├─┬ github-username@2.1.0 
│ │ │ └─┬ gh-got@2.4.0 
│ │ │   ├─┬ got@5.7.1 
│ │ │   │ ├─┬ create-error-class@3.0.2 
│ │ │   │ │ └── capture-stack-trace@1.0.0 
│ │ │   │ ├── duplexer2@0.1.4 
│ │ │   │ ├── is-redirect@1.0.0 
│ │ │   │ ├── is-retry-allowed@1.1.0 
│ │ │   │ ├── is-stream@1.1.0 
│ │ │   │ ├── lowercase-keys@1.0.1 
│ │ │   │ ├── node-status-codes@1.0.0 
│ │ │   │ ├── object-assign@4.1.1 
│ │ │   │ ├─┬ parse-json@2.2.0 
│ │ │   │ │ └─┬ error-ex@1.3.1 
│ │ │   │ │   └── is-arrayish@0.2.1 
│ │ │   │ ├── read-all-stream@3.1.0 
│ │ │   │ ├── timed-out@3.1.3 
│ │ │   │ ├── unzip-response@1.0.2 
│ │ │   │ └─┬ url-parse-lax@1.0.0 
│ │ │   │   └── prepend-http@1.0.4 
│ │ │   └── object-assign@4.1.1 
│ │ ├─┬ glob@7.1.2 
│ │ │ ├── fs.realpath@1.0.0 
│ │ │ ├─┬ inflight@1.0.6 
│ │ │ │ └── wrappy@1.0.2 
│ │ │ ├── inherits@2.0.3 
│ │ │ ├─┬ minimatch@3.0.4 
│ │ │ │ └─┬ brace-expansion@1.1.11 
│ │ │ │   ├── balanced-match@1.0.0 
│ │ │ │   └── concat-map@0.0.1 
│ │ │ └── once@1.4.0 
│ │ ├─┬ gruntfile-editor@1.2.1 
│ │ │ └─┬ ast-query@2.0.0 
│ │ │   ├─┬ acorn-jsx@3.0.1 
│ │ │   │ └── acorn@3.3.0 
│ │ │   ├─┬ escodegen-wallaby@1.6.18 
│ │ │   │ ├── esprima@2.7.3 
│ │ │   │ ├── estraverse@1.9.3 
│ │ │   │ ├── esutils@2.0.2 
│ │ │   │ ├─┬ optionator@0.8.2 
│ │ │   │ │ ├── deep-is@0.1.3 
│ │ │   │ │ ├── fast-levenshtein@2.0.6 
│ │ │   │ │ ├── levn@0.3.0 
│ │ │   │ │ ├── prelude-ls@1.1.2 
│ │ │   │ │ ├── type-check@0.3.2 
│ │ │   │ │ └── wordwrap@1.0.0 
│ │ │   │ └─┬ source-map@0.2.0 
│ │ │   │   └── amdefine@1.0.1 
│ │ │   └── traverse@0.6.6 
│ │ ├─┬ html-wiring@1.2.0 
│ │ │ ├─┬ cheerio@0.19.0 
│ │ │ │ ├─┬ css-select@1.0.0 
│ │ │ │ │ ├── boolbase@1.0.0 
│ │ │ │ │ ├── css-what@1.0.0 
│ │ │ │ │ ├── domutils@1.4.3 
│ │ │ │ │ └── nth-check@1.0.1 
│ │ │ │ ├─┬ dom-serializer@0.1.0 
│ │ │ │ │ └── domelementtype@1.1.3 
│ │ │ │ ├── entities@1.1.1 
│ │ │ │ ├─┬ htmlparser2@3.8.3 
│ │ │ │ │ ├── domelementtype@1.3.0 
│ │ │ │ │ ├── domhandler@2.3.0 
│ │ │ │ │ ├── domutils@1.5.1 
│ │ │ │ │ ├── entities@1.0.0 
│ │ │ │ │ └─┬ readable-stream@1.1.14 
│ │ │ │ │   ├── isarray@0.0.1 
│ │ │ │ │   └── string_decoder@0.10.31 
│ │ │ │ └── lodash@3.10.1 
│ │ │ └── detect-newline@1.0.3 
│ │ ├─┬ istextorbinary@2.2.1 
│ │ │ ├── binaryextensions@2.1.1 
│ │ │ ├── editions@1.3.4 
│ │ │ └── textextensions@2.2.0 
│ │ ├── lodash@4.17.10 
│ │ ├─┬ mem-fs-editor@2.3.0 
│ │ │ ├── commondir@1.0.1 
│ │ │ ├── deep-extend@0.4.2 
│ │ │ ├── ejs@2.6.1 
│ │ │ ├─┬ globby@4.1.0 
│ │ │ │ ├─┬ array-union@1.0.2 
│ │ │ │ │ └── array-uniq@1.0.3 
│ │ │ │ ├── arrify@1.0.1 
│ │ │ │ ├── glob@6.0.4 
│ │ │ │ ├── object-assign@4.1.1 
│ │ │ │ └── pify@2.3.0 
│ │ │ ├─┬ multimatch@2.1.0 
│ │ │ │ └── array-differ@1.0.0 
│ │ │ └─┬ vinyl@1.2.0 
│ │ │   ├── clone@1.0.4 
│ │ │   ├── clone-stats@0.0.1 
│ │ │   └── replace-ext@0.0.1 
│ │ ├─┬ mkdirp@0.5.1 
│ │ │ └── minimist@0.0.8 
│ │ ├─┬ nopt@3.0.6 
│ │ │ └── abbrev@1.1.1 
│ │ ├── path-exists@2.1.0 
│ │ ├── path-is-absolute@1.0.1 
│ │ ├── pretty-bytes@3.0.1 
│ │ ├── read-chunk@1.0.1 
│ │ ├─┬ read-pkg-up@1.0.1 
│ │ │ └─┬ read-pkg@1.1.0 
│ │ │   ├─┬ load-json-file@1.1.0 
│ │ │   │ ├── graceful-fs@4.1.11 
│ │ │   │ └─┬ strip-bom@2.0.0 
│ │ │   │   └── is-utf8@0.2.1 
│ │ │   └── path-type@1.1.0 
│ │ ├── rimraf@2.6.2 
│ │ ├─┬ run-async@2.3.0 
│ │ │ └── is-promise@2.1.0 
│ │ ├─┬ shelljs@0.7.8 
│ │ │ ├── interpret@1.1.0 
│ │ │ └─┬ rechoir@0.6.2 
│ │ │   └─┬ resolve@1.7.1 
│ │ │     └── path-parse@1.0.5 
│ │ ├── text-table@0.2.0 
│ │ ├─┬ through2@2.0.3 
│ │ │ └─┬ readable-stream@2.3.6 
│ │ │   ├── core-util-is@1.0.2 
│ │ │   ├── isarray@1.0.0 
│ │ │   ├── process-nextick-args@2.0.0 
│ │ │   └── string_decoder@1.1.1 
│ │ ├─┬ underscore.string@3.3.4 
│ │ │ ├── sprintf-js@1.1.1 
│ │ │ └── util-deprecate@1.0.2 
│ │ ├─┬ user-home@2.0.0 
│ │ │ └── os-homedir@1.0.2 
│ │ ├── yeoman-assert@2.2.3 
│ │ ├─┬ yeoman-environment@1.6.6 
│ │ │ ├─┬ chalk@1.1.3 
│ │ │ │ ├── ansi-styles@2.2.1 
│ │ │ │ └── supports-color@2.0.0 
│ │ │ ├── diff@2.2.3 
│ │ │ ├── grouped-queue@0.3.3 
│ │ │ ├─┬ inquirer@1.2.3 
│ │ │ │ ├── ansi-escapes@1.4.0 
│ │ │ │ ├─┬ chalk@1.1.3 
│ │ │ │ │ ├── ansi-styles@2.2.1 
│ │ │ │ │ └── supports-color@2.0.0 
│ │ │ │ ├─┬ cli-cursor@1.0.2 
│ │ │ │ │ └─┬ restore-cursor@1.0.1 
│ │ │ │ │   ├── exit-hook@1.1.1 
│ │ │ │ │   └── onetime@1.1.0 
│ │ │ │ ├── cli-width@2.2.0 
│ │ │ │ ├─┬ external-editor@1.1.1 
│ │ │ │ │ ├─┬ spawn-sync@1.0.15 
│ │ │ │ │ │ ├─┬ concat-stream@1.6.2 
│ │ │ │ │ │ │ ├── buffer-from@1.0.0 
│ │ │ │ │ │ │ └── typedarray@0.0.6 
│ │ │ │ │ │ └── os-shim@0.1.3 
│ │ │ │ │ └─┬ tmp@0.0.29 
│ │ │ │ │   └── os-tmpdir@1.0.2 
│ │ │ │ ├─┬ figures@1.7.0 
│ │ │ │ │ └── object-assign@4.1.1 
│ │ │ │ ├── mute-stream@0.0.6 
│ │ │ │ ├── rx@4.1.0 
│ │ │ │ └── through@2.3.8 
│ │ │ ├─┬ log-symbols@1.0.2 
│ │ │ │ └─┬ chalk@1.1.3 
│ │ │ │   ├── ansi-styles@2.2.1 
│ │ │ │   └── supports-color@2.0.0 
│ │ │ ├─┬ mem-fs@1.1.3 
│ │ │ │ └─┬ vinyl-file@2.0.0 
│ │ │ │   └─┬ strip-bom-stream@2.0.0 
│ │ │ │     └── first-chunk-stream@2.0.0 
│ │ │ └── untildify@2.1.0 
│ │ ├─┬ yeoman-test@1.7.2 
│ │ │ ├─┬ inquirer@5.2.0 
│ │ │ │ ├── ansi-escapes@3.1.0 
│ │ │ │ ├─┬ cli-cursor@2.1.0 
│ │ │ │ │ └─┬ restore-cursor@2.0.0 
│ │ │ │ │   └─┬ onetime@2.0.1 
│ │ │ │ │     └── mimic-fn@1.2.0 
│ │ │ │ ├─┬ external-editor@2.2.0 
│ │ │ │ │ ├── chardet@0.4.2 
│ │ │ │ │ ├─┬ iconv-lite@0.4.23 
│ │ │ │ │ │ └── safer-buffer@2.1.2 
│ │ │ │ │ └── tmp@0.0.33 
│ │ │ │ ├── figures@2.0.0 
│ │ │ │ ├── mute-stream@0.0.7 
│ │ │ │ ├─┬ rxjs@5.5.11 
│ │ │ │ │ └── symbol-observable@1.0.1 
│ │ │ │ ├─┬ string-width@2.1.1 
│ │ │ │ │ └── is-fullwidth-code-point@2.0.0 
│ │ │ │ └─┬ strip-ansi@4.0.0 
│ │ │ │   └── ansi-regex@3.0.0 
│ │ │ ├─┬ sinon@5.0.10 
│ │ │ │ ├─┬ @sinonjs/formatio@2.0.0 
│ │ │ │ │ └── samsam@1.3.0 
│ │ │ │ ├── diff@3.5.0 
│ │ │ │ ├── lodash.get@4.4.2 
│ │ │ │ ├── lolex@2.7.0 
│ │ │ │ ├─┬ nise@1.3.3 
│ │ │ │ │ ├── just-extend@1.1.27 
│ │ │ │ │ ├─┬ path-to-regexp@1.7.0 
│ │ │ │ │ │ └── isarray@0.0.1 
│ │ │ │ │ └── text-encoding@0.6.4 
│ │ │ │ └── type-detect@4.0.8 
│ │ │ ├─┬ yeoman-environment@2.1.1 
│ │ │ │ ├─┬ cross-spawn@6.0.5 
│ │ │ │ │ ├── nice-try@1.0.4 
│ │ │ │ │ ├── path-key@2.0.1 
│ │ │ │ │ └─┬ shebang-command@1.2.0 
│ │ │ │ │   └── shebang-regex@1.0.0 
│ │ │ │ ├── debug@3.1.0 
│ │ │ │ ├── diff@3.5.0 
│ │ │ │ ├─┬ globby@8.0.1 
│ │ │ │ │ ├─┬ dir-glob@2.0.0 
│ │ │ │ │ │ └─┬ path-type@3.0.0 
│ │ │ │ │ │   └── pify@3.0.0 
│ │ │ │ │ ├─┬ fast-glob@2.2.2 
│ │ │ │ │ │ ├─┬ @mrmlnc/readdir-enhanced@2.2.1 
│ │ │ │ │ │ │ ├── call-me-maybe@1.0.1 
│ │ │ │ │ │ │ └── glob-to-regexp@0.3.0 
│ │ │ │ │ │ ├── @nodelib/fs.stat@1.1.0 
│ │ │ │ │ │ ├─┬ glob-parent@3.1.0 
│ │ │ │ │ │ │ ├── is-glob@3.1.0 
│ │ │ │ │ │ │ └── path-dirname@1.0.2 
│ │ │ │ │ │ ├─┬ is-glob@4.0.0 
│ │ │ │ │ │ │ └── is-extglob@2.1.1 
│ │ │ │ │ │ ├── merge2@1.2.2 
│ │ │ │ │ │ └─┬ micromatch@3.1.10 
│ │ │ │ │ │   ├── arr-diff@4.0.0 
│ │ │ │ │ │   ├── array-unique@0.3.2 
│ │ │ │ │ │   ├─┬ braces@2.3.2 
│ │ │ │ │ │   │ ├── arr-flatten@1.1.0 
│ │ │ │ │ │   │ ├─┬ extend-shallow@2.0.1 
│ │ │ │ │ │   │ │ └── is-extendable@0.1.1 
│ │ │ │ │ │   │ ├─┬ fill-range@4.0.0 
│ │ │ │ │ │   │ │ ├── extend-shallow@2.0.1 
│ │ │ │ │ │   │ │ ├─┬ is-number@3.0.0 
│ │ │ │ │ │   │ │ │ └─┬ kind-of@3.2.2 
│ │ │ │ │ │   │ │ │   └── is-buffer@1.1.6 
│ │ │ │ │ │   │ │ ├── repeat-string@1.6.1 
│ │ │ │ │ │   │ │ └── to-regex-range@2.1.1 
│ │ │ │ │ │   │ ├── isobject@3.0.1 
│ │ │ │ │ │   │ ├── repeat-element@1.1.2 
│ │ │ │ │ │   │ ├─┬ snapdragon-node@2.1.1 
│ │ │ │ │ │   │ │ ├─┬ define-property@1.0.0 
│ │ │ │ │ │   │ │ │ └─┬ is-descriptor@1.0.2 
│ │ │ │ │ │   │ │ │   ├── is-accessor-descriptor@1.0.0 
│ │ │ │ │ │   │ │ │   └── is-data-descriptor@1.0.0 
│ │ │ │ │ │   │ │ └─┬ snapdragon-util@3.0.1 
│ │ │ │ │ │   │ │   └── kind-of@3.2.2 
│ │ │ │ │ │   │ └── split-string@3.1.0 
│ │ │ │ │ │   ├─┬ define-property@2.0.2 
│ │ │ │ │ │   │ └─┬ is-descriptor@1.0.2 
│ │ │ │ │ │   │   ├── is-accessor-descriptor@1.0.0 
│ │ │ │ │ │   │   └── is-data-descriptor@1.0.0 
│ │ │ │ │ │   ├─┬ extend-shallow@3.0.2 
│ │ │ │ │ │   │ ├── assign-symbols@1.0.0 
│ │ │ │ │ │   │ └─┬ is-extendable@1.0.1 
│ │ │ │ │ │   │   └── is-plain-object@2.0.4 
│ │ │ │ │ │   ├─┬ extglob@2.0.4 
│ │ │ │ │ │   │ ├─┬ define-property@1.0.0 
│ │ │ │ │ │   │ │ └─┬ is-descriptor@1.0.2 
│ │ │ │ │ │   │ │   ├── is-accessor-descriptor@1.0.0 
│ │ │ │ │ │   │ │   └── is-data-descriptor@1.0.0 
│ │ │ │ │ │   │ ├─┬ expand-brackets@2.1.4 
│ │ │ │ │ │   │ │ ├── define-property@0.2.5 
│ │ │ │ │ │   │ │ ├── extend-shallow@2.0.1 
│ │ │ │ │ │   │ │ └── posix-character-classes@0.1.1 
│ │ │ │ │ │   │ └── extend-shallow@2.0.1 
│ │ │ │ │ │   ├─┬ fragment-cache@0.2.1 
│ │ │ │ │ │   │ └── map-cache@0.2.2 
│ │ │ │ │ │   ├── kind-of@6.0.2 
│ │ │ │ │ │   ├─┬ nanomatch@1.2.9 
│ │ │ │ │ │   │ ├─┬ is-odd@2.0.0 
│ │ │ │ │ │   │ │ └── is-number@4.0.0 
│ │ │ │ │ │   │ └── is-windows@1.0.2 
│ │ │ │ │ │   ├── object.pick@1.3.0 
│ │ │ │ │ │   ├─┬ regex-not@1.0.2 
│ │ │ │ │ │   │ └─┬ safe-regex@1.1.0 
│ │ │ │ │ │   │   └── ret@0.1.15 
│ │ │ │ │ │   ├─┬ snapdragon@0.8.2 
│ │ │ │ │ │   │ ├─┬ base@0.11.2 
│ │ │ │ │ │   │ │ ├─┬ cache-base@1.0.1 
│ │ │ │ │ │   │ │ │ ├─┬ collection-visit@1.0.0 
│ │ │ │ │ │   │ │ │ │ ├── map-visit@1.0.0 
│ │ │ │ │ │   │ │ │ │ └── object-visit@1.0.1 
│ │ │ │ │ │   │ │ │ ├── get-value@2.0.6 
│ │ │ │ │ │   │ │ │ ├─┬ has-value@1.0.0 
│ │ │ │ │ │   │ │ │ │ └─┬ has-values@1.0.0 
│ │ │ │ │ │   │ │ │ │   └── kind-of@4.0.0 
│ │ │ │ │ │   │ │ │ ├─┬ set-value@2.0.0 
│ │ │ │ │ │   │ │ │ │ └── extend-shallow@2.0.1 
│ │ │ │ │ │   │ │ │ ├─┬ to-object-path@0.3.0 
│ │ │ │ │ │   │ │ │ │ └── kind-of@3.2.2 
│ │ │ │ │ │   │ │ │ ├─┬ union-value@1.0.0 
│ │ │ │ │ │   │ │ │ │ └─┬ set-value@0.4.3 
│ │ │ │ │ │   │ │ │ │   └── extend-shallow@2.0.1 
│ │ │ │ │ │   │ │ │ └─┬ unset-value@1.0.0 
│ │ │ │ │ │   │ │ │   └─┬ has-value@0.3.1 
│ │ │ │ │ │   │ │ │     ├── has-values@0.1.4 
│ │ │ │ │ │   │ │ │     └── isobject@2.1.0 
│ │ │ │ │ │   │ │ ├─┬ class-utils@0.3.6 
│ │ │ │ │ │   │ │ │ ├── arr-union@3.1.0 
│ │ │ │ │ │   │ │ │ ├── define-property@0.2.5 
│ │ │ │ │ │   │ │ │ └─┬ static-extend@0.1.2 
│ │ │ │ │ │   │ │ │   ├── define-property@0.2.5 
│ │ │ │ │ │   │ │ │   └─┬ object-copy@0.1.0 
│ │ │ │ │ │   │ │ │     ├── copy-descriptor@0.1.1 
│ │ │ │ │ │   │ │ │     ├── define-property@0.2.5 
│ │ │ │ │ │   │ │ │     └── kind-of@3.2.2 
│ │ │ │ │ │   │ │ ├── component-emitter@1.2.1 
│ │ │ │ │ │   │ │ ├─┬ define-property@1.0.0 
│ │ │ │ │ │   │ │ │ └─┬ is-descriptor@1.0.2 
│ │ │ │ │ │   │ │ │   ├── is-accessor-descriptor@1.0.0 
│ │ │ │ │ │   │ │ │   └── is-data-descriptor@1.0.0 
│ │ │ │ │ │   │ │ ├─┬ mixin-deep@1.3.1 
│ │ │ │ │ │   │ │ │ ├── for-in@1.0.2 
│ │ │ │ │ │   │ │ │ └── is-extendable@1.0.1 
│ │ │ │ │ │   │ │ └── pascalcase@0.1.1 
│ │ │ │ │ │   │ ├─┬ define-property@0.2.5 
│ │ │ │ │ │   │ │ └─┬ is-descriptor@0.1.6 
│ │ │ │ │ │   │ │   ├─┬ is-accessor-descriptor@0.1.6 
│ │ │ │ │ │   │ │   │ └── kind-of@3.2.2 
│ │ │ │ │ │   │ │   ├─┬ is-data-descriptor@0.1.4 
│ │ │ │ │ │   │ │   │ └── kind-of@3.2.2 
│ │ │ │ │ │   │ │   └── kind-of@5.1.0 
│ │ │ │ │ │   │ ├── extend-shallow@2.0.1 
│ │ │ │ │ │   │ ├── source-map@0.5.7 
│ │ │ │ │ │   │ ├─┬ source-map-resolve@0.5.2 
│ │ │ │ │ │   │ │ ├── atob@2.1.1 
│ │ │ │ │ │   │ │ ├── decode-uri-component@0.2.0 
│ │ │ │ │ │   │ │ ├── resolve-url@0.2.1 
│ │ │ │ │ │   │ │ ├── source-map-url@0.4.0 
│ │ │ │ │ │   │ │ └── urix@0.1.0 
│ │ │ │ │ │   │ └── use@3.1.0 
│ │ │ │ │ │   └── to-regex@3.0.2 
│ │ │ │ │ ├── ignore@3.3.8 
│ │ │ │ │ ├── pify@3.0.0 
│ │ │ │ │ └── slash@1.0.0 
│ │ │ │ ├─┬ is-scoped@1.0.0 
│ │ │ │ │ └── scoped-regex@1.0.0 
│ │ │ │ ├── log-symbols@2.2.0 
│ │ │ │ └── untildify@3.0.3 
│ │ │ └─┬ yeoman-generator@2.0.5 
│ │ │   ├── dargs@5.1.0 
│ │ │   ├── dateformat@3.0.3 
│ │ │   ├─┬ find-up@2.1.0 
│ │ │   │ └─┬ locate-path@2.0.0 
│ │ │   │   ├─┬ p-locate@2.0.0 
│ │ │   │   │ └─┬ p-limit@1.2.0 
│ │ │   │   │   └── p-try@1.0.0 
│ │ │   │   └── path-exists@3.0.0 
│ │ │   ├─┬ github-username@4.1.0 
│ │ │   │ └─┬ gh-got@6.0.0 
│ │ │   │   ├─┬ got@7.1.0 
│ │ │   │   │ ├─┬ decompress-response@3.3.0 
│ │ │   │   │ │ └── mimic-response@1.0.0 
│ │ │   │   │ ├── duplexer3@0.1.4 
│ │ │   │   │ ├── get-stream@3.0.0 
│ │ │   │   │ ├─┬ isurl@1.0.0 
│ │ │   │   │ │ ├─┬ has-to-string-tag-x@1.4.1 
│ │ │   │   │ │ │ └── has-symbol-support-x@1.4.2 
│ │ │   │   │ │ └── is-object@1.0.1 
│ │ │   │   │ ├── p-cancelable@0.3.0 
│ │ │   │   │ ├─┬ p-timeout@1.2.1 
│ │ │   │   │ │ └── p-finally@1.0.0 
│ │ │   │   │ ├── timed-out@4.0.1 
│ │ │   │   │ └── url-to-options@1.0.1 
│ │ │   │   └── is-plain-obj@1.1.0 
│ │ │   ├─┬ make-dir@1.3.0 
│ │ │   │ └── pify@3.0.0 
│ │ │   ├─┬ mem-fs-editor@4.0.2 
│ │ │   │ ├── deep-extend@0.5.1 
│ │ │   │ ├── isbinaryfile@3.0.2 
│ │ │   │ └─┬ vinyl@2.1.0 
│ │ │   │   ├── clone@2.1.1 
│ │ │   │   ├── clone-buffer@1.0.0 
│ │ │   │   ├── clone-stats@1.0.0 
│ │ │   │   ├── cloneable-readable@1.1.2 
│ │ │   │   ├── remove-trailing-separator@1.1.0 
│ │ │   │   └── replace-ext@1.0.0 
│ │ │   ├── pretty-bytes@4.0.2 
│ │ │   ├── read-chunk@2.1.0 
│ │ │   ├─┬ read-pkg-up@3.0.0 
│ │ │   │ └─┬ read-pkg@3.0.0 
│ │ │   │   ├─┬ load-json-file@4.0.0 
│ │ │   │   │ ├─┬ parse-json@4.0.0 
│ │ │   │   │ │ └── json-parse-better-errors@1.0.2 
│ │ │   │   │ └── strip-bom@3.0.0 
│ │ │   │   └── path-type@3.0.0 
│ │ │   └── shelljs@0.8.2 
│ │ └─┬ yeoman-welcome@1.0.1 
│ │   └─┬ chalk@1.1.3 
│ │     ├── ansi-styles@2.2.1 
│ │     └── supports-color@2.0.0 
│ └─┬ yosay@2.0.2 
│   ├── ansi-regex@2.1.1 
│   ├─┬ chalk@1.1.3 
│   │ ├── ansi-styles@2.2.1 
│   │ └── supports-color@2.0.0 
│   ├── cli-boxes@1.0.0 
│   ├── pad-component@0.0.1 
│   ├─┬ string-width@2.1.1 
│   │ ├── is-fullwidth-code-point@2.0.0 
│   │ └─┬ strip-ansi@4.0.0 
│   │   └── ansi-regex@3.0.0 
│   ├── strip-ansi@3.0.1 
│   ├─┬ taketalk@1.0.0 
│   │ └── minimist@1.2.0 
│   └─┬ wrap-ansi@2.1.0 
│     └─┬ string-width@1.0.2 
│       ├── code-point-at@1.1.0 
│       └── is-fullwidth-code-point@1.0.0 
└─┬ yo@2.0.2 
  ├── async@2.6.1 
  ├─┬ chalk@1.1.3 
  │ ├── ansi-styles@2.2.1 
  │ ├── escape-string-regexp@1.0.5 
  │ ├── has-ansi@2.0.0 
  │ ├── strip-ansi@3.0.1 
  │ └── supports-color@2.0.0 
  ├── cli-list@0.2.0 
  ├─┬ configstore@3.1.2 
  │ ├─┬ dot-prop@4.2.0 
  │ │ └── is-obj@1.0.1 
  │ ├── graceful-fs@4.1.11 
  │ ├── make-dir@1.3.0 
  │ ├─┬ unique-string@1.0.0 
  │ │ └── crypto-random-string@1.0.0 
  │ ├─┬ write-file-atomic@2.3.0 
  │ │ ├── imurmurhash@0.1.4 
  │ │ └── signal-exit@3.0.2 
  │ └── xdg-basedir@3.0.0 
  ├─┬ cross-spawn@5.1.0 
  │ ├─┬ lru-cache@4.1.3 
  │ │ ├── pseudomap@1.0.2 
  │ │ └── yallist@2.1.2 
  │ ├─┬ shebang-command@1.2.0 
  │ │ └── shebang-regex@1.0.0 
  │ └─┬ which@1.3.1 
  │   └── isexe@2.0.0 
  ├── figures@2.0.0 
  ├─┬ fullname@3.3.0 
  │ ├─┬ execa@0.6.3 
  │ │ ├── is-stream@1.1.0 
  │ │ ├── npm-run-path@2.0.2 
  │ │ ├── p-finally@1.0.0 
  │ │ └── strip-eof@1.0.0 
  │ ├── filter-obj@1.1.0 
  │ ├─┬ mem@1.1.0 
  │ │ └── mimic-fn@1.2.0 
  │ ├─┬ p-any@1.1.0 
  │ │ └─┬ p-some@2.0.1 
  │ │   └─┬ aggregate-error@1.0.0 
  │ │     ├── clean-stack@1.3.0 
  │ │     └── indent-string@3.2.0 
  │ ├── p-try@1.0.0 
  │ ├─┬ passwd-user@2.1.0 
  │ │ ├─┬ execa@0.4.0 
  │ │ │ ├── cross-spawn-async@2.2.5 
  │ │ │ ├── npm-run-path@1.0.0 
  │ │ │ └── path-key@1.0.0 
  │ │ └── pify@2.3.0 
  │ └─┬ rc@1.2.8 
  │   ├── deep-extend@0.6.0 
  │   ├── ini@1.3.5 
  │   └── strip-json-comments@2.0.1 
  ├─┬ global-tunnel-ng@2.1.1 
  │ └── tunnel@0.0.4 
  ├─┬ got@8.3.1 
  │ ├── @sindresorhus/is@0.7.0 
  │ ├─┬ cacheable-request@2.1.4 
  │ │ ├── clone-response@1.0.2 
  │ │ ├── http-cache-semantics@3.8.1 
  │ │ ├─┬ keyv@3.0.0 
  │ │ │ └── json-buffer@3.0.0 
  │ │ ├── lowercase-keys@1.0.0 
  │ │ ├─┬ normalize-url@2.0.1 
  │ │ │ ├─┬ query-string@5.1.1 
  │ │ │ │ ├── decode-uri-component@0.2.0 
  │ │ │ │ └── strict-uri-encode@1.1.0 
  │ │ │ └── sort-keys@2.0.0 
  │ │ └── responselike@1.0.2 
  │ ├── decompress-response@3.3.0 
  │ ├── duplexer3@0.1.4 
  │ ├── get-stream@3.0.0 
  │ ├─┬ into-stream@3.1.0 
  │ │ ├─┬ from2@2.3.0 
  │ │ │ ├── inherits@2.0.3 
  │ │ │ └─┬ readable-stream@2.3.6 
  │ │ │   ├── core-util-is@1.0.2 
  │ │ │   ├── isarray@1.0.0 
  │ │ │   ├── process-nextick-args@2.0.0 
  │ │ │   ├── string_decoder@1.1.1 
  │ │ │   └── util-deprecate@1.0.2 
  │ │ └── p-is-promise@1.1.0 
  │ ├── is-retry-allowed@1.1.0 
  │ ├─┬ isurl@1.0.0 
  │ │ ├─┬ has-to-string-tag-x@1.4.1 
  │ │ │ └── has-symbol-support-x@1.4.2 
  │ │ └── is-object@1.0.1 
  │ ├── lowercase-keys@1.0.1 
  │ ├── mimic-response@1.0.0 
  │ ├── p-cancelable@0.4.1 
  │ ├── p-timeout@2.0.1 
  │ ├── pify@3.0.0 
  │ ├── safe-buffer@5.1.2 
  │ ├── timed-out@4.0.1 
  │ ├─┬ url-parse-lax@3.0.0 
  │ │ └── prepend-http@2.0.0 
  │ └── url-to-options@1.0.1 
  ├─┬ humanize-string@1.0.2 
  │ └── decamelize@1.2.0 
  ├─┬ inquirer@3.3.0 
  │ ├── ansi-escapes@3.1.0 
  │ ├─┬ chalk@2.4.1 
  │ │ ├── ansi-styles@3.2.1 
  │ │ └─┬ supports-color@5.4.0 
  │ │   └── has-flag@3.0.0 
  │ ├─┬ cli-cursor@2.1.0 
  │ │ └─┬ restore-cursor@2.0.0 
  │ │   └── onetime@2.0.1 
  │ ├── cli-width@2.2.0 
  │ ├─┬ external-editor@2.2.0 
  │ │ ├── chardet@0.4.2 
  │ │ ├─┬ iconv-lite@0.4.23 
  │ │ │ └── safer-buffer@2.1.2 
  │ │ └── tmp@0.0.33 
  │ ├── mute-stream@0.0.7 
  │ ├─┬ run-async@2.3.0 
  │ │ └── is-promise@2.1.0 
  │ ├── rx-lite@4.0.8 
  │ ├── rx-lite-aggregates@4.0.8 
  │ ├─┬ string-width@2.1.1 
  │ │ ├── is-fullwidth-code-point@2.0.0 
  │ │ └─┬ strip-ansi@4.0.0 
  │ │   └── ansi-regex@3.0.0 
  │ ├─┬ strip-ansi@4.0.0 
  │ │ └── ansi-regex@3.0.0 
  │ └── through@2.3.8 
  ├─┬ insight@0.8.4 
  │ ├── async@1.5.2 
  │ ├─┬ configstore@1.4.0 
  │ │ ├── os-tmpdir@1.0.2 
  │ │ ├── osenv@0.1.5 
  │ │ ├── uuid@2.0.3 
  │ │ ├─┬ write-file-atomic@1.3.4 
  │ │ │ └── slide@1.1.6 
  │ │ └── xdg-basedir@2.0.0 
  │ ├─┬ inquirer@0.10.1 
  │ │ ├── ansi-escapes@1.4.0 
  │ │ ├─┬ cli-cursor@1.0.2 
  │ │ │ └─┬ restore-cursor@1.0.1 
  │ │ │   ├── exit-hook@1.1.1 
  │ │ │   └── onetime@1.1.0 
  │ │ ├── cli-width@1.1.1 
  │ │ ├── figures@1.7.0 
  │ │ ├── lodash@3.10.1 
  │ │ ├─┬ readline2@1.0.1 
  │ │ │ ├── code-point-at@1.1.0 
  │ │ │ ├─┬ is-fullwidth-code-point@1.0.0 
  │ │ │ │ └── number-is-nan@1.0.1 
  │ │ │ └── mute-stream@0.0.5 
  │ │ ├─┬ run-async@0.1.0 
  │ │ │ └─┬ once@1.4.0 
  │ │ │   └── wrappy@1.0.2 
  │ │ └── rx-lite@3.1.2 
  │ ├─┬ lodash.debounce@3.1.1 
  │ │ └── lodash._getnative@3.9.1 
  │ ├── object-assign@4.1.1 
  │ ├─┬ os-name@1.0.3 
  │ │ ├── osx-release@1.1.0 
  │ │ └── win-release@1.1.1 
  │ ├─┬ request@2.87.0 
  │ │ ├── aws-sign2@0.7.0 
  │ │ ├── aws4@1.7.0 
  │ │ ├── caseless@0.12.0 
  │ │ ├─┬ combined-stream@1.0.6 
  │ │ │ └── delayed-stream@1.0.0 
  │ │ ├── extend@3.0.1 
  │ │ ├── forever-agent@0.6.1 
  │ │ ├─┬ form-data@2.3.2 
  │ │ │ └── asynckit@0.4.0 
  │ │ ├─┬ har-validator@5.0.3 
  │ │ │ ├─┬ ajv@5.5.2 
  │ │ │ │ ├── co@4.6.0 
  │ │ │ │ ├── fast-deep-equal@1.1.0 
  │ │ │ │ ├── fast-json-stable-stringify@2.0.0 
  │ │ │ │ └── json-schema-traverse@0.3.1 
  │ │ │ └── har-schema@2.0.0 
  │ │ ├─┬ http-signature@1.2.0 
  │ │ │ ├── assert-plus@1.0.0 
  │ │ │ ├─┬ jsprim@1.4.1 
  │ │ │ │ ├── extsprintf@1.3.0 
  │ │ │ │ ├── json-schema@0.2.3 
  │ │ │ │ └── verror@1.10.0 
  │ │ │ └─┬ sshpk@1.14.1 
  │ │ │   ├── asn1@0.2.3 
  │ │ │   ├── bcrypt-pbkdf@1.0.1 
  │ │ │   ├── dashdash@1.14.1 
  │ │ │   ├── ecc-jsbn@0.1.1 
  │ │ │   ├── getpass@0.1.7 
  │ │ │   ├── jsbn@0.1.1 
  │ │ │   └── tweetnacl@0.14.5 
  │ │ ├── is-typedarray@1.0.0 
  │ │ ├── isstream@0.1.2 
  │ │ ├── json-stringify-safe@5.0.1 
  │ │ ├─┬ mime-types@2.1.18 
  │ │ │ └── mime-db@1.33.0 
  │ │ ├── oauth-sign@0.8.2 
  │ │ ├── performance-now@2.1.0 
  │ │ ├── qs@6.5.2 
  │ │ └── tunnel-agent@0.6.0 
  │ ├─┬ tough-cookie@2.3.4 
  │ │ └── punycode@1.4.1 
  │ └── uuid@3.2.1 
  ├── lodash@4.17.10 
  ├─┬ meow@3.7.0 
  │ ├─┬ camelcase-keys@2.1.0 
  │ │ └── camelcase@2.1.1 
  │ ├─┬ loud-rejection@1.6.0 
  │ │ └─┬ currently-unhandled@0.4.1 
  │ │   └── array-find-index@1.0.2 
  │ ├── map-obj@1.0.1 
  │ ├── minimist@1.2.0 
  │ ├─┬ normalize-package-data@2.4.0 
  │ │ ├── hosted-git-info@2.6.0 
  │ │ ├─┬ is-builtin-module@1.0.0 
  │ │ │ └── builtin-modules@1.1.1 
  │ │ └─┬ validate-npm-package-license@3.0.3 
  │ │   ├─┬ spdx-correct@3.0.0 
  │ │   │ └── spdx-license-ids@3.0.0 
  │ │   └─┬ spdx-expression-parse@3.0.0 
  │ │     └── spdx-exceptions@2.1.0 
  │ ├─┬ read-pkg-up@1.0.1 
  │ │ ├─┬ find-up@1.1.2 
  │ │ │ └── path-exists@2.1.0 
  │ │ └─┬ read-pkg@1.1.0 
  │ │   ├─┬ load-json-file@1.1.0 
  │ │   │ └── pify@2.3.0 
  │ │   └─┬ path-type@1.1.0 
  │ │     └── pify@2.3.0 
  │ ├─┬ redent@1.0.0 
  │ │ ├─┬ indent-string@2.1.0 
  │ │ │ └─┬ repeating@2.0.1 
  │ │ │   └── is-finite@1.0.2 
  │ │ └── strip-indent@1.0.1 
  │ └── trim-newlines@1.0.0 
  ├─┬ npm-keyword@5.0.0 
  │ ├─┬ got@7.1.0 
  │ │ ├── is-plain-obj@1.1.0 
  │ │ ├── p-cancelable@0.3.0 
  │ │ ├── p-timeout@1.2.1 
  │ │ └─┬ url-parse-lax@1.0.0 
  │ │   └── prepend-http@1.0.4 
  │ └── registry-url@3.1.0 
  ├─┬ opn@4.0.2 
  │ └─┬ pinkie-promise@2.0.1 
  │   └── pinkie@2.0.4 
  ├─┬ package-json@4.0.1 
  │ ├─┬ got@6.7.1 
  │ │ ├─┬ create-error-class@3.0.2 
  │ │ │ └── capture-stack-trace@1.0.0 
  │ │ ├── is-redirect@1.0.0 
  │ │ ├── unzip-response@2.0.1 
  │ │ └─┬ url-parse-lax@1.0.0 
  │ │   └── prepend-http@1.0.4 
  │ ├── registry-auth-token@3.3.2 
  │ └── semver@5.5.0 
  ├─┬ parse-help@0.1.1 
  │ └─┬ execall@1.0.0 
  │   └─┬ clone-regexp@1.0.1 
  │     ├── is-regexp@1.0.0 
  │     └── is-supported-regexp-flag@1.0.1 
  ├─┬ read-pkg-up@2.0.0 
  │ ├─┬ find-up@2.1.0 
  │ │ └─┬ locate-path@2.0.0 
  │ │   ├─┬ p-locate@2.0.0 
  │ │   │ └── p-limit@1.2.0 
  │ │   └── path-exists@3.0.0 
  │ └─┬ read-pkg@2.0.0 
  │   ├─┬ load-json-file@2.0.0 
  │   │ ├─┬ parse-json@2.2.0 
  │   │ │ └─┬ error-ex@1.3.1 
  │   │ │   └── is-arrayish@0.2.1 
  │   │ ├── pify@2.3.0 
  │   │ └── strip-bom@3.0.0 
  │   └── path-type@2.0.0 
  ├─┬ root-check@1.0.0 
  │ ├─┬ downgrade-root@1.2.2 
  │ │ ├── default-uid@1.0.0 
  │ │ └── is-root@1.0.0 
  │ └─┬ sudo-block@1.2.0 
  │   └── is-docker@1.1.0 
  ├─┬ sort-on@2.0.0 
  │ └── arrify@1.0.1 
  ├── string-length@1.0.1 
  ├─┬ tabtab@1.3.2 
  │ ├─┬ debug@2.6.9 
  │ │ └── ms@2.0.0 
  │ ├─┬ inquirer@1.2.3 
  │ │ ├── ansi-escapes@1.4.0 
  │ │ ├─┬ cli-cursor@1.0.2 
  │ │ │ └─┬ restore-cursor@1.0.1 
  │ │ │   └── onetime@1.1.0 
  │ │ ├─┬ external-editor@1.1.1 
  │ │ │ ├─┬ spawn-sync@1.0.15 
  │ │ │ │ ├─┬ concat-stream@1.6.2 
  │ │ │ │ │ ├── buffer-from@1.0.0 
  │ │ │ │ │ └── typedarray@0.0.6 
  │ │ │ │ └── os-shim@0.1.3 
  │ │ │ └── tmp@0.0.29 
  │ │ ├── figures@1.7.0 
  │ │ ├── mute-stream@0.0.6 
  │ │ ├── rx@4.1.0 
  │ │ └─┬ string-width@1.0.2 
  │ │   └── is-fullwidth-code-point@1.0.0 
  │ ├─┬ mkdirp@0.5.1 
  │ │ └── minimist@0.0.8 
  │ └─┬ npmlog@2.0.4 
  │   ├── ansi@0.3.1 
  │   ├─┬ are-we-there-yet@1.1.5 
  │   │ └── delegates@1.0.0 
  │   └─┬ gauge@1.2.7 
  │     ├── has-unicode@2.0.1 
  │     ├── lodash.pad@4.5.1 
  │     ├── lodash.padend@4.6.1 
  │     └── lodash.padstart@4.6.1 
  ├── titleize@1.0.1 
  ├─┬ update-notifier@2.5.0 
  │ ├─┬ boxen@1.3.0 
  │ │ ├── ansi-align@2.0.0 
  │ │ ├── camelcase@4.1.0 
  │ │ ├─┬ chalk@2.4.1 
  │ │ │ ├── ansi-styles@3.2.1 
  │ │ │ └── supports-color@5.4.0 
  │ │ ├─┬ term-size@1.2.0 
  │ │ │ └── execa@0.7.0 
  │ │ └── widest-line@2.0.0 
  │ ├─┬ chalk@2.4.1 
  │ │ ├── ansi-styles@3.2.1 
  │ │ └── supports-color@5.4.0 
  │ ├── import-lazy@2.1.0 
  │ ├─┬ is-ci@1.1.0 
  │ │ └── ci-info@1.1.3 
  │ ├─┬ is-installed-globally@0.1.0 
  │ │ ├── global-dirs@0.1.1 
  │ │ └─┬ is-path-inside@1.0.1 
  │ │   └── path-is-inside@1.0.2 
  │ ├── is-npm@1.0.0 
  │ ├── latest-version@3.1.0 
  │ └── semver-diff@2.1.0 
  ├─┬ user-home@2.0.0 
  │ └── os-homedir@1.0.2 
  ├─┬ yeoman-character@1.1.0 
  │ └─┬ supports-color@3.2.3 
  │   └── has-flag@1.0.0 
  ├─┬ yeoman-doctor@2.1.0 
  │ ├─┬ bin-version-check@2.1.0 
  │ │ ├─┬ bin-version@1.0.4 
  │ │ │ └─┬ find-versions@1.2.1 
  │ │ │   └── semver-regex@1.0.0 
  │ │ ├── semver@4.3.6 
  │ │ └── semver-truncate@1.1.2 
  │ ├─┬ each-async@1.1.1 
  │ │ ├── onetime@1.1.0 
  │ │ └── set-immediate-shim@1.0.1 
  │ ├── log-symbols@1.0.2 
  │ ├── object-values@1.0.0 
  │ └─┬ twig@0.8.9 
  │   ├─┬ minimatch@3.0.4 
  │   │ └─┬ brace-expansion@1.1.11 
  │   │   ├── balanced-match@1.0.0 
  │   │   └── concat-map@0.0.1 
  │   └─┬ walk@2.3.13 
  │     └── foreachasync@3.0.0 
  ├─┬ yeoman-environment@2.1.1 
  │ ├─┬ chalk@2.4.1 
  │ │ ├── ansi-styles@3.2.1 
  │ │ └── supports-color@5.4.0 
  │ ├─┬ cross-spawn@6.0.5 
  │ │ ├── nice-try@1.0.4 
  │ │ └── path-key@2.0.1 
  │ ├── debug@3.1.0 
  │ ├── diff@3.5.0 
  │ ├─┬ globby@8.0.1 
  │ │ ├─┬ array-union@1.0.2 
  │ │ │ └── array-uniq@1.0.3 
  │ │ ├─┬ dir-glob@2.0.0 
  │ │ │ └── path-type@3.0.0 
  │ │ ├─┬ fast-glob@2.2.2 
  │ │ │ ├─┬ @mrmlnc/readdir-enhanced@2.2.1 
  │ │ │ │ ├── call-me-maybe@1.0.1 
  │ │ │ │ └── glob-to-regexp@0.3.0 
  │ │ │ ├── @nodelib/fs.stat@1.1.0 
  │ │ │ ├─┬ glob-parent@3.1.0 
  │ │ │ │ ├─┬ is-glob@3.1.0 
  │ │ │ │ │ └── is-extglob@2.1.1 
  │ │ │ │ └── path-dirname@1.0.2 
  │ │ │ ├─┬ is-glob@4.0.0 
  │ │ │ │ └── is-extglob@2.1.1 
  │ │ │ ├── merge2@1.2.2 
  │ │ │ └─┬ micromatch@3.1.10 
  │ │ │   ├── arr-diff@4.0.0 
  │ │ │   ├── array-unique@0.3.2 
  │ │ │   ├─┬ braces@2.3.2 
  │ │ │   │ ├── arr-flatten@1.1.0 
  │ │ │   │ ├─┬ extend-shallow@2.0.1 
  │ │ │   │ │ └── is-extendable@0.1.1 
  │ │ │   │ ├─┬ fill-range@4.0.0 
  │ │ │   │ │ ├─┬ extend-shallow@2.0.1 
  │ │ │   │ │ │ └── is-extendable@0.1.1 
  │ │ │   │ │ ├─┬ is-number@3.0.0 
  │ │ │   │ │ │ └─┬ kind-of@3.2.2 
  │ │ │   │ │ │   └── is-buffer@1.1.6 
  │ │ │   │ │ ├── repeat-string@1.6.1 
  │ │ │   │ │ └── to-regex-range@2.1.1 
  │ │ │   │ ├── isobject@3.0.1 
  │ │ │   │ ├── repeat-element@1.1.2 
  │ │ │   │ ├─┬ snapdragon-node@2.1.1 
  │ │ │   │ │ ├── define-property@1.0.0 
  │ │ │   │ │ └─┬ snapdragon-util@3.0.1 
  │ │ │   │ │   └─┬ kind-of@3.2.2 
  │ │ │   │ │     └── is-buffer@1.1.6 
  │ │ │   │ └── split-string@3.1.0 
  │ │ │   ├─┬ define-property@2.0.2 
  │ │ │   │ └─┬ is-descriptor@1.0.2 
  │ │ │   │   ├── is-accessor-descriptor@1.0.0 
  │ │ │   │   └── is-data-descriptor@1.0.0 
  │ │ │   ├─┬ extend-shallow@3.0.2 
  │ │ │   │ ├── assign-symbols@1.0.0 
  │ │ │   │ └─┬ is-extendable@1.0.1 
  │ │ │   │   └── is-plain-object@2.0.4 
  │ │ │   ├─┬ extglob@2.0.4 
  │ │ │   │ ├── define-property@1.0.0 
  │ │ │   │ ├─┬ expand-brackets@2.1.4 
  │ │ │   │ │ ├─┬ define-property@0.2.5 
  │ │ │   │ │ │ └─┬ is-descriptor@0.1.6 
  │ │ │   │ │ │   └── kind-of@5.1.0 
  │ │ │   │ │ ├─┬ extend-shallow@2.0.1 
  │ │ │   │ │ │ └── is-extendable@0.1.1 
  │ │ │   │ │ └── posix-character-classes@0.1.1 
  │ │ │   │ └─┬ extend-shallow@2.0.1 
  │ │ │   │   └── is-extendable@0.1.1 
  │ │ │   ├─┬ fragment-cache@0.2.1 
  │ │ │   │ └── map-cache@0.2.2 
  │ │ │   ├── kind-of@6.0.2 
  │ │ │   ├─┬ nanomatch@1.2.9 
  │ │ │   │ ├─┬ is-odd@2.0.0 
  │ │ │   │ │ └── is-number@4.0.0 
  │ │ │   │ └── is-windows@1.0.2 
  │ │ │   ├── object.pick@1.3.0 
  │ │ │   ├─┬ regex-not@1.0.2 
  │ │ │   │ └─┬ safe-regex@1.1.0 
  │ │ │   │   └── ret@0.1.15 
  │ │ │   ├─┬ snapdragon@0.8.2 
  │ │ │   │ ├─┬ base@0.11.2 
  │ │ │   │ │ ├─┬ cache-base@1.0.1 
  │ │ │   │ │ │ ├─┬ collection-visit@1.0.0 
  │ │ │   │ │ │ │ ├── map-visit@1.0.0 
  │ │ │   │ │ │ │ └── object-visit@1.0.1 
  │ │ │   │ │ │ ├── get-value@2.0.6 
  │ │ │   │ │ │ ├─┬ has-value@1.0.0 
  │ │ │   │ │ │ │ └─┬ has-values@1.0.0 
  │ │ │   │ │ │ │   └─┬ kind-of@4.0.0 
  │ │ │   │ │ │ │     └── is-buffer@1.1.6 
  │ │ │   │ │ │ ├─┬ set-value@2.0.0 
  │ │ │   │ │ │ │ ├── extend-shallow@2.0.1 
  │ │ │   │ │ │ │ └── is-extendable@0.1.1 
  │ │ │   │ │ │ ├─┬ to-object-path@0.3.0 
  │ │ │   │ │ │ │ └─┬ kind-of@3.2.2 
  │ │ │   │ │ │ │   └── is-buffer@1.1.6 
  │ │ │   │ │ │ ├─┬ union-value@1.0.0 
  │ │ │   │ │ │ │ ├── is-extendable@0.1.1 
  │ │ │   │ │ │ │ └─┬ set-value@0.4.3 
  │ │ │   │ │ │ │   └── extend-shallow@2.0.1 
  │ │ │   │ │ │ └─┬ unset-value@1.0.0 
  │ │ │   │ │ │   └─┬ has-value@0.3.1 
  │ │ │   │ │ │     ├── has-values@0.1.4 
  │ │ │   │ │ │     └── isobject@2.1.0 
  │ │ │   │ │ ├─┬ class-utils@0.3.6 
  │ │ │   │ │ │ ├── arr-union@3.1.0 
  │ │ │   │ │ │ ├─┬ define-property@0.2.5 
  │ │ │   │ │ │ │ └─┬ is-descriptor@0.1.6 
  │ │ │   │ │ │ │   └── kind-of@5.1.0 
  │ │ │   │ │ │ └─┬ static-extend@0.1.2 
  │ │ │   │ │ │   ├─┬ define-property@0.2.5 
  │ │ │   │ │ │   │ └─┬ is-descriptor@0.1.6 
  │ │ │   │ │ │   │   └── kind-of@5.1.0 
  │ │ │   │ │ │   └─┬ object-copy@0.1.0 
  │ │ │   │ │ │     ├── copy-descriptor@0.1.1 
  │ │ │   │ │ │     ├─┬ define-property@0.2.5 
  │ │ │   │ │ │     │ └─┬ is-descriptor@0.1.6 
  │ │ │   │ │ │     │   └── kind-of@5.1.0 
  │ │ │   │ │ │     └─┬ kind-of@3.2.2 
  │ │ │   │ │ │       └── is-buffer@1.1.6 
  │ │ │   │ │ ├── component-emitter@1.2.1 
  │ │ │   │ │ ├── define-property@1.0.0 
  │ │ │   │ │ ├─┬ mixin-deep@1.3.1 
  │ │ │   │ │ │ └── for-in@1.0.2 
  │ │ │   │ │ └── pascalcase@0.1.1 
  │ │ │   │ ├─┬ define-property@0.2.5 
  │ │ │   │ │ └─┬ is-descriptor@0.1.6 
  │ │ │   │ │   ├─┬ is-accessor-descriptor@0.1.6 
  │ │ │   │ │   │ └─┬ kind-of@3.2.2 
  │ │ │   │ │   │   └── is-buffer@1.1.6 
  │ │ │   │ │   ├─┬ is-data-descriptor@0.1.4 
  │ │ │   │ │   │ └─┬ kind-of@3.2.2 
  │ │ │   │ │   │   └── is-buffer@1.1.6 
  │ │ │   │ │   └── kind-of@5.1.0 
  │ │ │   │ ├─┬ extend-shallow@2.0.1 
  │ │ │   │ │ └── is-extendable@0.1.1 
  │ │ │   │ ├── source-map@0.5.7 
  │ │ │   │ ├─┬ source-map-resolve@0.5.2 
  │ │ │   │ │ ├── atob@2.1.1 
  │ │ │   │ │ ├── resolve-url@0.2.1 
  │ │ │   │ │ ├── source-map-url@0.4.0 
  │ │ │   │ │ └── urix@0.1.0 
  │ │ │   │ └── use@3.1.0 
  │ │ │   └── to-regex@3.0.2 
  │ │ ├─┬ glob@7.1.2 
  │ │ │ ├── fs.realpath@1.0.0 
  │ │ │ ├── inflight@1.0.6 
  │ │ │ └── path-is-absolute@1.0.1 
  │ │ ├── ignore@3.3.8 
  │ │ └── slash@1.0.0 
  │ ├── grouped-queue@0.3.3 
  │ ├─┬ inquirer@5.2.0 
  │ │ └─┬ rxjs@5.5.11 
  │ │   └── symbol-observable@1.0.1 
  │ ├─┬ is-scoped@1.0.0 
  │ │ └── scoped-regex@1.0.0 
  │ ├── log-symbols@2.2.0 
  │ ├─┬ mem-fs@1.1.3 
  │ │ ├─┬ through2@2.0.3 
  │ │ │ └── xtend@4.0.1 
  │ │ ├─┬ vinyl@1.2.0 
  │ │ │ ├── clone@1.0.4 
  │ │ │ ├── clone-stats@0.0.1 
  │ │ │ └── replace-ext@0.0.1 
  │ │ └─┬ vinyl-file@2.0.0 
  │ │   ├── pify@2.3.0 
  │ │   ├─┬ strip-bom@2.0.0 
  │ │   │ └── is-utf8@0.2.1 
  │ │   └─┬ strip-bom-stream@2.0.0 
  │ │     └── first-chunk-stream@2.0.0 
  │ ├─┬ strip-ansi@4.0.0 
  │ │ └── ansi-regex@3.0.0 
  │ ├── text-table@0.2.0 
  │ └── untildify@3.0.3 
  └─┬ yosay@2.0.2 
    ├── ansi-regex@2.1.1 
    ├─┬ ansi-styles@3.2.1 
    │ └─┬ color-convert@1.9.1 
    │   └── color-name@1.1.3 
    ├── cli-boxes@1.0.0 
    ├── pad-component@0.0.1 
    ├─┬ taketalk@1.0.0 
    │ └── get-stdin@4.0.1 
    └─┬ wrap-ansi@2.1.0 
      └─┬ string-width@1.0.2 
        └── is-fullwidth-code-point@1.0.0 
```
