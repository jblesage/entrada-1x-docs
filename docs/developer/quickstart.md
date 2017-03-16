# Quickstart Guide

This Quickstart guide is designed as a resource for PHP developers to learn how to structure and create new modules within the Entrada Platform using our latest development standards and best practices.

## History of Entrada

The history of Entrada development goes back over a decade and as time has passed, best practices in both the industry and our codebase have improved. These improvements change the way that we write our code. Even though the way that we write our code has improved we have not, in many cases, gone back and updated older modules within the application.

While this can cause some confusion and perhaps frustration for new Entrada developers, it is important to realize that updating older modules in the application would potentially cause significant merge conflicts for schools that have customized those modules to meet their specific needs. The time taken to resolve these conflicts would be time taken away from the development of new functionality to support the mission of the instituions.

Instead of a large rewrite of the core codebase the Entrada Development Community in 2016 committed to moving ahead with a transition to Service Oriented Architecture using the Lumen micro framework. This transition will begin with Entrada ME 1.9 and this documentation will be updated accordingly to reflect these changes.

## Creating a New Module

### Entrada ME 1.8

This documention will walk you through step-by-step creating a new module that we are calling "Sandbox" in Entrada ME 1.8. This new Sandbox module will consist of the following:

1. A new **migration** to accommodate the new database table and ACL rules.
2. A new **model** that will be used to access the new table.
3. A new **public module** that will show the learners a list of all of the sandboxes.
4. A new **admin module** that will allow administrative users to add, edit, and delete the sandboxes.
5. A new **view helper** that will render the form used on both the add and edit pages, and another that will render a simple sidebar item.

Pay special attention to note of the bolded words above: migration, model, public module, admin module, and view helper. This terminology is frequently used by Entrada developers.

----
>All files that are references within the document can be [downloaded here](https://github.com/EntradaProject/entrada-1x-docs/raw/master/resources/quickstart/Entrada-ME-1.8-Sandbox-Quickstart.tar.gz) for review.

----

### Let's Begin

Open the `~/Sites/entrada-1x-me.dev` folder using PhpStorm or whatever IDE/Editor you have selected. Ensure that you set the tab size in your editor to 4 spaces as the tab character will not be accepted. For more information on coding standards visit the [Coding Standards](standards/) section.

#### 1. Create a New Database Migration

This new Sandbox module is going to need a new table in the `entrada` database. In order to systematically and consistently apply these types of changes you would create a migration. 

A migration is used to record and apply changes and data transformations within the Entrada databases. For more information on how to use migrations please see the [Database documentation](/developer/database) section.

To create a new migration run the following command in terminal within the Entrada root folder:

    php entrada migrate --create

You will be asked to provide the corresponding GitHub Issue number, and then a new file will be generated within the `www-root/core/library/Migrate` folder that looks like `2017_01_11_150030_1477.php`. This file will have `up()`, `down()`, and `audit()` methods that you must complete.

    <?php
    class Migrate_2017_01_11_150030_1477 extends Entrada_Cli_Migrate {
    
        /**
         * Required: SQL / PHP that performs the upgrade migration.
         */
        public function up() {
            $this->record();
            ?>
            CREATE TABLE IF NOT EXISTS `sandbox` (
            `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
            `title` varchar(128) NOT NULL DEFAULT '',
            `description` text,
            `created_date` bigint(64) unsigned DEFAULT NULL,
            `created_by` int(11) unsigned DEFAULT NULL,
            `updated_date` bigint(64) unsigned DEFAULT NULL,
            `updated_by` int(11) unsigned DEFAULT NULL,
            `deleted_date` bigint(64) unsigned DEFAULT NULL,
            `deleted_by` int(11) unsigned DEFAULT NULL,
            PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
    
            INSERT INTO `<?php echo AUTH_DATABASE; ?>`.`acl_permissions` (`permission_id`, `resource_type`, `resource_value`, `entity_type`, `entity_value`, `app_id`, `create`, `read`, `update`, `delete`, `assertion`)
            VALUES
            (NULL, 'sandbox', NULL, NULL, NULL, 1, NULL, 1, NULL, NULL, 'NotGuest'),
            (NULL, 'sandbox', NULL, 'group:role', 'staff:admin', 1, 1, NULL, 1, 1, NULL);
            <?php
            $this->stop();
    
            return $this->run();
        }
    
        /**
         * Required: SQL / PHP that performs the downgrade migration.
         */
        public function down() {
            $this->record();
            ?>
            DROP TABLE `sandbox`;
    
            DELETE FROM `<?php echo AUTH_DATABASE; ?>`.`acl_permissions` WHERE `resource_type` = 'sandbox';
            <?php
            $this->stop();
    
            return $this->run();
        }
    
        /**
         * Optional: PHP that verifies whether or not the changes outlined
         * in "up" are present in the active database.
         *
         * Return Values: -1 (not run) | 0 (changes not present or complete) | 1 (present)
         *
         * @return int
         */
        public function audit() {
            $migration = new Models_Migration();
            if ($migration->tableExists(DATABASE_NAME, "sandbox")) {
                return 1;
            }
    
            return 0;
        }
    }

Once your new migration file has been created, you can apply and test these changes by executing:

    php entrada migrate --up

#### 2. Create the New Model

You will use the Eloquent `Sandbox` model class exclusively to access the new `entrada.sandbox` table created by the migration. You can create a new file `www-root/core/api/app/Models/Entrada/Sandbox.php`. This file will generate the methods necessary to access that table.

    <?php

    namespace App\Models\Entrada;

    use Auth;
    use App;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\SoftDeletes;

    class Sandbox extends Model 
    {
        use SoftDeletes;

        const CREATED_AT = 'created_date';
        const UPDATED_AT = 'updated_date';
        const DELETED_AT = 'deleted_date';
        
        /**
         * The table associated with the model.
         *
         * @var string
         */
        protected $table = 'sandbox';

        /**
         * The storage format of the model's date columns.
         *
         * @var string
         */
        protected $dateFormat = 'U';

        /**
         * The attributes that should be mutated to dates.
         *
         * @var array
         */
        protected $dates = ['created_date', 'updated_date','deleted_date'];

        /**
         * The attributes that are mass assignable.
         *
         * @var array
         */
        protected $fillable = ['title', 'description'];

        /**
         * The user who created this sandbox
         */
        public function created_by() {
            return $this->belongsTo('App\Models\Auth\User', 'created_by');
        }

        /**
         * The user who updated this sandbox
         */
        public function updated_by() {
            return $this->belongsTo('App\Models\Auth\User', 'updated_by');
        }

        public static function boot()
         {
            parent::boot();

            /**
             * Set fields on creating event
             */
            static::creating(function($model)
            {
                $user = Auth::user();          
                $model->created_by = $user->id;
                $model->updated_by = $user->id;
            });

            /**
             * Set owner of sandbox upon creation
             */
            static::created(function($model)
            {
                $user = Auth::user();
                $model->users()->attach($user->id);
            });

            /**
             * Set fields on updating event
             */
            static::updating(function($model)
            {
                $user = Auth::user();
                $model->updated_by = $user->id;
            });

            /**
             * Set fields on deleting event if not force deleting
             */
            static::deleting(function($model)
            {
                $user = Auth::user();

                if (!$model->isForceDeleting()) {
                  $model->deleted_by = $user->id;
                  $model->save();
                }
            });
        }
        
    }

Once that is created, you will be able to access and manipulate data of that model using Eloquent commands. [Click here to view the different ways you can query the model using Eloquent.](https://laravel.com/docs/5.4/eloquent)

#### 3. Attach an ACLPolicy to the Model

In order for us to determine if a user can perform a certain action, we need to use ACL. Entrada_ACL already exists, but Laravel/Lumen also has an ACL system called ```Gate```. Gate provides us with a number of convenience and security features, so we will want to integrate both together using an ACLPolicy. 

The standard ```www-root/core/api/app/Policies/ACLPolicy``` can apply to most models, but if you need custom functionality, you can create one in the same folder.

To attach the standard ACLPolicy to your Model, open up ```www-root/core/api/app/Providers/AuthServiceProvider``` and insert the following line into your boot() method: 

    Gate::policy(Sandbox::class, ACLPolicy::class);

(Make sure to use the appropriate namespaces at the top of the file). Here is an example of the full file: 

    <?php

    namespace App\Providers;

    use Auth;
    use App\Models\Auth\User;
    use Illuminate\Support\Facades\Gate;
    use Illuminate\Support\ServiceProvider;
    use App\Auth\RegisteredAppUserProvider;
    use App\Auth\LocalUserProvider;
    use App\Auth\SsoUserProvider;
    use App\Auth\LdapUserProvider;
    use App\Policies\ACLPolicy;

    use App\Models\Entrada\Sandbox;

    class AuthServiceProvider extends ServiceProvider
    {
        /**
         * Register the service provider.
         *
         * @return void
         */
        public function register()
        {

            // Register new user providers when functionality cannot 
            // be handled entirely by Eloquent or Database user providers

            Auth::provider('registered_apps', function ($app, $config) {
                return new RegisteredAppUserProvider($this->createHasher($config['hasher']), $config['model']);
            });

            Auth::provider('local', function ($app, $config) {
                return new LocalUserProvider($this->createHasher($config['hasher']), $config['model']);
            });

            Auth::provider('sso', function ($app, $config) {
                return new SsoUserProvider($this->createHasher($config['hasher']), $config['model']);
            });

            Auth::provider('ldap', function ($app, $config) {
                return new LdapUserProvider();
            });
        }

        /**
         * Boot the authentication services for the application.
         *
         * @return void
         */
        public function boot() {

            // Register the ACL Policy for the Sandbox model. 
            // Do this for all models that require ACL access.

            Gate::policy(Sandbox::class, ACLPolicy::class);

        }

        /**
         * Create a new instance of the hasher.
         *
         * @return \Illuminate\Contracts\Hashing\Hasher
         */
        protected function createHasher($hasher)
        {
            $class = '\\'.ltrim($hasher, '\\');

            return new $class;
        }
    }

#### 3. Create a new controller

Once you have your model, you can now start creating the different routes that the client will access to view and manipulate the data.

Create a new controller file at `/home/jblesage/Sites/entrada-1x-me/www-root/core/api/app/Http/SandboxController.php`. Here is an example:

    <?php

    namespace App\Http\Controllers;

    use Auth;
    use App\Models\Entrada\Sandbox;
    use Illuminate\Http\Request;

    class SandboxController extends Controller
    {

        public function __construct()
        {
            $this->middleware('auth');
            $this->middleware('jwt.refreshWhenExpired');

            $this->input_fields = [
                'title' => 'required|string',
                'description' => 'required|string',
            ];
        }

        /**
         * Display a listing of the resource.
         *
         * @param  \App\Models\Entrada\Sandbox  $sandbox
         * @return \Illuminate\Http\Response
         */
        public function index(Sandbox $sandbox)
        {
            $this->authorize('view', $sandbox);

            return [
                'sandboxes' => $sandbox->with('created_by', 'updated_by')->orderBy('created_date', 'desc')->paginate(),
                'current_user_can' => [
                    'read' => Auth::user()->can('view', $sandbox),
                    'create' => Auth::user()->can('create', $sandbox),
                    'update' => Auth::user()->can('update', $sandbox),
                    'delete' => Auth::user()->can('delete', $sandbox),
                ]
            ];
        }

        /**
         * Store a newly created resource in storage.
         *
         * @param  \Illuminate\Http\Request     $request
         * @param  \App\Models\Entrada\Sandbox  $sandbox
         * @return \Illuminate\Http\Response
         */
        public function store(Request $request, Sandbox $sandbox)
        {
            // Authorizes the creation of sandbox
            $this->authorize('create', $sandbox);

            $this->validate($request, $this->input_fields);

            $new = $sandbox->create($request->only('title', 'description'));

            return response($new, 201);
        }

        /**
         * Display the specified resource.
         *
         * @param  \App\Models\Entrada\Sandbox  $sandbox
         * @param  int  $id
         * @return \Illuminate\Http\Response
         */
        public function show(Sandbox $sandbox, $id)
        {
            $this->authorize('view', $sandbox);

            return $sandbox->findOrFail($id);
        }

        /**
         * Update the specified resource in storage.
         * 
         * Note: PUT and PATCH methods in Laravel require
         * an extra header: "Content-Type: application/x-www-form-urlencoded"
         *
         * @param  \Illuminate\Http\Request  $request
         * @param  int  $id
         * @return \Illuminate\Http\Response
         */
        public function update(Request $request, $id)
        {
            $sandbox = Sandbox::findOrFail($id);

            // Authorizes the update of sandbox
            $this->authorize('update', $sandbox);

            // Validate request
            $this->validate($request, $this->input_fields);

            // Save new data to sandbox model
            $update = $sandbox->update($request->only('title', 'description'));

            return response($sandbox->findOrFail($id), 200);
        }

        /**
         * Remove the specified resource from storage.
         *
         * @param  \App\Models\Entrada\Sandbox  $sandbox
         * @param  int  $id
         * @return \Illuminate\Http\Response
         */
        public function destroy(Sandbox $sandbox, $id)
        {
            $this->authorize('delete', $sandbox);

            $delete = Sandbox::destroy($id);

            if ($delete)
            {
                // Successful delete returns a 204
                return response('', 204);
            }

            return response('', 404);
        }
    }

Let's look at each method in detail to learn more about how this works:

**1. The constructor**

    public function __construct()
    {
        $this->middleware('auth');
        $this->middleware('jwt.refreshWhenExpired');

        $this->input_fields = [
            'title' => 'required|string',
            'description' => 'required|string',
        ];
    }

The constructor, as in any class, is where we can set data that will be shared among the class methods, as well as loading up middleware. 

Any standard controller should use both of these middleware:

* **$this->middleware('auth')**

Since we JWT as our authentication method, the role of auth middleware is to determine: 

1. Is there a Bearer token attached to the request?
2. Is the token current?
3. Is the token attached to the current PHP session?

It may seem odd to be using JWT to attach to a session, but this was a design decision to be able to access existing features like Entrada_ACL, which rely on session variables to determine the capabilities of the logged-in user.

* **$this->middleware('jwt.refreshWhenExpired')**

The role of this middleware is to automatically attach a new JWT to the response if the current token has expired. Later on in this guide, we'll see how to capture this token for seamless use on the client side.

**2. The index() method**
    
    /**
     * Display a listing of the resource.
     *
     * @param  \App\Models\Entrada\Sandbox  $sandbox
     * @return \Illuminate\Http\Response
     */
    public function index(Sandbox $sandbox)
    {
        $this->authorize('view', $sandbox);

        return [
            'sandboxes' => $sandbox->with('created_by', 'updated_by')->orderBy('created_date', 'desc')->paginate(),
            'current_user_can' => [
                'read' => Auth::user()->can('view', $sandbox),
                'create' => Auth::user()->can('create', $sandbox),
                'update' => Auth::user()->can('update', $sandbox),
                'delete' => Auth::user()->can('delete', $sandbox),
            ]
        ];
    }

The index method is typically where we would want to get a list of paginated results from the model we are querying. As a best practice, you also want to include what capabilities are available to the user, as an easy way for the client to determine whether to display certain buttons.

When we call ```$this->authorize('view', $sandbox)```, we are actually calling the view() method in ACLPolicy, which in turn requests the "read" capability from ENTRADA_ACL.

When returning an array as the response, the framework will automatically convert it to JSON.

More info: 

- [Pagination](https://laravel.com/docs/5.4/pagination)
- [Authorization](https://laravel.com/docs/5.4/authorization)

**3. The store() method**

    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request     $request
     * @param  \App\Models\Entrada\Sandbox  $sandbox
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request, Sandbox $sandbox)
    {
        // Authorizes the creation of sandbox
        $this->authorize('create', $sandbox);

        $this->validate($request, $this->input_fields);

        $new = $sandbox->create($request->only('title', 'description'));

        return response($new, 201);
    }

Here is the standard route name for when we want to create a Model. 

After calling the authorize method, we want to validate the input data from the request using the `$this->validate($request, $this->input_fields)`. The validation fields are an associative array in the structure of `"input name" => "validation types"`. Validation types are separated with a pipe character. If the input variables do not pass validation, the request automatically fails.

After passing validation, we create the sandbox using the create() method. We inject the input values using the $request->only() method -- an associative array of keys and values. This is a clean, succinct way of passing only specific values to the Model.

Upon successful creation, we respond with the HTTP-compliant 201 response.

More info: 

- [Input validation](https://laravel.com/docs/5.4/validation)
- [Eloquent "Create" method](https://laravel.com/docs/5.4/eloquent-relationships#the-create-method)

**4. The show() method**

    /**
     * Display the specified resource.
     *
     * @param  \App\Models\Entrada\Sandbox  $sandbox
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function show(Sandbox $sandbox, $id)
    {
        $this->authorize('view', $sandbox);

        return $sandbox->findOrFail($id);
    }

The show() method is the standard Laravel verb to describe displaying one singular item. 

After authorizing the User to view the resource, we use the `findOrFail($id)` method to get the resource. What is great about this method is that it will either return the JSON representation of the object and a response of 200, or a 404 if none found.

More info: 

- [Retrieving Single Models](https://laravel.com/docs/5.4/eloquent#retrieving-single-models)

**5. The update() method**

    /**
     * Update the specified resource in storage.
     * 
     * Note: PUT and PATCH methods in Laravel require
     * an extra header: "Content-Type: application/x-www-form-urlencoded"
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, $id)
    {
        $sandbox = Sandbox::findOrFail($id);

        // Authorizes the update of sandbox
        $this->authorize('update', $sandbox);

        // Validate request
        $this->validate($request, $this->input_fields);

        // Save new data to sandbox model
        $update = $sandbox->update($request->only('title', 'description'));

        return response($sandbox->findOrFail($id), 200);
    }

The update() method, as the name implies, allows us to update an existing resource.

We start by reusing the findOrFail() method to make sure a resource exists. 

After finding a resource, authorizing the user, and validating data, we are able to update the field, this time again using the request->only method.

More info: 

- [Eloquent update()](https://laravel.com/docs/5.4/eloquent#updates)

**6. The destroy() method**

    /**
     * Remove the specified resource from storage.
     *
     * @param  \App\Models\Entrada\Sandbox  $sandbox
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function destroy(Sandbox $sandbox, $id)
    {
        $this->authorize('delete', $sandbox);

        $delete = Sandbox::destroy($id);

        if ($delete)
        {
            // Successful delete returns a 204
            return response('', 204);
        }

        return response('', 404);
    }

After authorizing the user to delete an instance, go ahead and delete the sandbox using the destroy() method. 

Notice that we use the HTTP-recommended response of a 204. That response code does not allow any content within the body of the response, and in this case it is also unnecessary. 

A 404 will be returned if the item did not exist before the deletion request occurred.

If you notice the Model file we set up earlier, this will result in a soft delete, because we included the use of the `SoftDeletes` trait. 

More info: 

- [Eloquent destroy()](https://laravel.com/docs/5.4/eloquent#deleting-models)

#### 4. Create the new Admin Module

Entrada does not use a typical MVC pattern as you would recognize from frameworks like Laravel, CakePHP, or Symfony. Although similiar in effect, our pattern is not entirely object oriented; we use a mix of object oriented and procedural design patterns.

Entrada's `.htaccess` file contains `mod_rewrite` rules that pipe the majority of requests through the `index.php` and `admin.php` files. The `index.php` file is used to display **public modules**, where as the `admin.php` file is used to display **admin modules**.

One significant difference between most MVC frameworks and Entrada is that our request routing is entirely automatic vs. being manually defined in a routes file. For example, if you visit [http://entrada-1x-me.dev/profile/gradebook/assignments](http://entrada-1x-me.dev/profile/gradebook/assignments) in your browser, Entrada will be loading the `www-root/core/modules/public/profile/gradebook/assignments/index.inc.php` file.
 
For a more thorough overview of this information please see the [Directory Structure](overview/#directory-structure) section within Overview. 

While the admin module example in this tutorial is quite simple (there was only a single file), you can create a much more extensible hierarchy by creating components within your modules. Here is an example of the Sandbox modules administrative file structure, if you wanted to have multiple URLs for your module:

    www-root/core/modules/admin/
        \_ sandbox.inc.php
            \_ sandbox/add.inc.php
            \_ sandbox/delete.inc.php
            \_ sandbox/edit.inc.php
            \_ sandbox/index.inc.php

However, in this case, we will be creating a single-page app with VueJS, so just the single `www-root/core/modules/admin/sandbox.inc.php` file will be necessary:

    <?php
    /**
     * Entrada [ http://www.entrada-project.org ]
     *
     * @author Matt Simpson <simpson@queensu.ca>
     * @copyright Copyright 2016 Queen's University. All Rights Reserved.
     */

    if (!defined("PARENT_INCLUDED")) {
        exit;
    } elseif (!isset($_SESSION["isAuthorized"]) || !(bool) $_SESSION["isAuthorized"]) {
        header("Location: " . ENTRADA_URL);
        exit;
    } elseif (!$ENTRADA_ACL->amIAllowed("sandbox", "update")) {
        add_error("Your account does not have the permissions required to use this module.<br /><br />If you believe you are receiving this message in error please contact <a href=\"mailto:" . html_encode($AGENT_CONTACTS["administrator"]["email"]) . "\">" . html_encode($AGENT_CONTACTS["administrator"]["name"]) . "</a> for assistance.");

        echo display_error();

        application_log("error", "Group [" . $ENTRADA_USER->getActiveGroup() . "] and role [" . $ENTRADA_USER->getActiveRole() . "] do not have access to this module [" . $MODULE . "]");
    } else {
        $PAGE_META["title"] = "Admin: Sandbox";
        $PAGE_META["description"] = "";
        $PAGE_META["keywords"] = "";

        $BREADCRUMB[] = array("url" => ENTRADA_RELATIVE . "/admin/sandbox", "title" => "Admin Side: Sandbox");

        $HEAD[]   = '<link rel="stylesheet" href="'.ENTRADA_URL.'/css/sandbox.css">';
        $JQUERY[] = '<script src="https://unpkg.com/vue@2.2.1"></script>';
        $JQUERY[] = '<script src="https://unpkg.com/axios@0.15.3/dist/axios.min.js"></script>';

        $sidebar = new Views_Sandbox_Sidebar();
        $sidebar->render();
        ?>
        <div id="module-sandbox">

          <div class="pagination clearfix">
            <button type="button" data-toggle="modal" data-target="#create-sandbox" class="btn btn-primary pull-left" v-if="current_user_can.create">
              <?php echo $translate->_("Create New Sandbox"); ?>
            </button>

            <p class="text-center summary">
              <?php echo $translate->_("Page {{ sandboxes.current_page }} / {{ sandboxes.last_page }} &mdash; {{ sandboxes.total }} Sandboxes Found"); ?>
            </p>

            <ul class="pull-right">
              <li v-bind:class="{ disabled:!sandboxes.prev_page_url }">
                <a href="#" aria-label="Previous" title="Previous" v-on:click.prevent="sandboxes.prev_page_url ? getSandboxes(sandboxes.prev_page_url) : null">
                  <span aria-hidden="true">«</span>
                </a>
              </li>
              <li v-bind:class="{ disabled:!sandboxes.next_page_url }">
                <a href="#" aria-label="Next" v-on:click.prevent="sandboxes.next_page_url ? getSandboxes(sandboxes.next_page_url) : null">
                  <span aria-hidden="true">»</span>
                </a>
              </li>
            </ul>
          </div>

          <div class="table-responsive">
            <table class="table table-borderless">
              <tr>
                <th width="150"><?php echo $translate->_("Title"); ?></th>
                <th width="200"><?php echo $translate->_("Description"); ?></th>
                <th><?php echo $translate->_("Created By"); ?></th>
                <th><?php echo $translate->_("Updated By"); ?></th>
                <th><?php echo $translate->_("Actions"); ?></th>
              </tr>
              <tr v-for="sandbox in sandboxes.data">
                <td>{{ sandbox.title }}</td>
                <td>{{ sandbox.description }}</td>
                <td><span v-if="sandbox.created_by">{{ sandbox.created_by.firstname }} {{ sandbox.created_by.lastname }}</span></td>
                <td><span v-if="sandbox.updated_by">{{ sandbox.updated_by.firstname }} {{ sandbox.updated_by.lastname }}</span></td>
                <td>
                  <button class="edit-modal btn btn-warning" v-if="current_user_can.update" v-on:click.prevent="editSandbox(sandbox)">
                    <span class="glyphicon glyphicon-edit"></span> <?php echo $translate->_("Edit"); ?>
                  </button>
                  <button class="edit-modal btn btn-danger" v-if="current_user_can.delete" v-on:click.prevent="deleteSandbox(sandbox.id)">
                    <span class="glyphicon glyphicon-trash"></span> <?php echo $translate->_("Delete"); ?>
                  </button>
                </td>
              </tr>
            </table>
          </div>

          <?php

          // Create sandbox modal
          $create_sandbox_modal = new Views_Gradebook_Modal([
            "id" => "create-sandbox",
            "title" => $translate->_("Create New Sandbox"),
          ]);

          $create_sandbox_modal->setBody('
            <form class="form-horizontal" v-on:submit.prevent="createSandbox">
              <div class="control-group" v-bind:class="{ error: form_errors.title }">
                <label class="control-label" for="newTitle">'.$translate->_("Title").'</label>
                <div class="controls">
                  <input type="text" id="newTitle" placeholder="'.$translate->_("Title").'" v-model="new_sandbox.title">
                  <p v-if="form_errors.title" class="error text-error">
                    {{ form_errors.title }}
                  </p>
                </div>
              </div>
              <div class="control-group" v-bind:class="{ error: form_errors.description }">
                <label class="control-label" for="newDescription">'.$translate->_("Description").'</label>
                <div class="controls">
                  <textarea id="newDescription" placeholder="'.$translate->_("Description").'" v-model="new_sandbox.description"></textarea>
                  <p v-if="form_errors.description" class="error text-error">
                    {{ form_errors.description }}
                  </p>
                </div>
              </div>

              <div class="control-group">
                <button type="submit" class="btn btn-success pull-right">'.$translate->_("Create Sandbox").'</button>
              </div>
            </form>
          ');

          $create_sandbox_modal->render();

          // Edit sandbox modal
          $edit_sandbox_modal = new Views_Gradebook_Modal([
            "id" => "edit-sandbox",
            "title" => $translate->_("Edit")." ".'"{{ update_sandbox.title }}"',
          ]);

          $edit_sandbox_modal->setBody('
            <form class="form-horizontal" v-on:submit.prevent="updateSandbox(update_sandbox.id)">
              <div class="control-group" v-bind:class="{ error: form_errors.title }">
                <label class="control-label" for="updateTitle">'.$translate->_("Title").'</label>
                <div class="controls">
                  <input type="text" id="updateTitle" placeholder="'.$translate->_("Title").'" v-model="update_sandbox.title">
                  <p v-if="form_errors.title" class="error text-error">
                    {{ form_errors.title }}
                  </p>
                </div>
              </div>
              <div class="control-group" v-bind:class="{ error: form_errors.description }">
                <label class="control-label" for="updateDescription">'.$translate->_("Description").'</label>
                <div class="controls">
                  <textarea id="updateDescription" placeholder="'.$translate->_("Description").'" v-model="update_sandbox.description"></textarea>
                  <p v-if="form_errors.description" class="error text-error">
                    {{ form_errors.description }}
                  </p>
                </div>
              </div>

              <div class="control-group">
                <button type="submit" class="btn btn-success pull-right">'.$translate->_("Update Sandbox").'</button>
              </div>
            </form>
          ');

          $edit_sandbox_modal->render();

          ?>
            
        </div>
        <?php

        // sandbox.js needs to run after DOM content
        echo '<script src="'.ENTRADA_URL.'/javascript/sandbox.js?release='.html_encode(APPLICATION_VERSION).'"></script>';
    }

Let's go through the different parts of this file: 

**File Header**

    <?php
    /**
     * Entrada [ http://www.entrada-project.org ]
     *
     * @author Matt Simpson <simpson@queensu.ca>
     * @copyright Copyright 2016 Queen's University. All Rights Reserved.
     */

    if (!defined("PARENT_INCLUDED")) {
        exit;
    } elseif (!isset($_SESSION["isAuthorized"]) || !(bool) $_SESSION["isAuthorized"]) {
        header("Location: " . ENTRADA_URL);
        exit;
    } elseif (!$ENTRADA_ACL->amIAllowed("sandbox", "update")) {
        add_error("Your account does not have the permissions required to use this module.<br /><br />If you believe you are receiving this message in error please contact <a href=\"mailto:" . html_encode($AGENT_CONTACTS["administrator"]["email"]) . "\">" . html_encode($AGENT_CONTACTS["administrator"]["name"]) . "</a> for assistance.");

        echo display_error();

        application_log("error", "Group [" . $ENTRADA_USER->getActiveGroup() . "] and role [" . $ENTRADA_USER->getActiveRole() . "] do not have access to this module [" . $MODULE . "]");
    } else {
        $PAGE_META["title"] = "Admin: Sandbox";
        $PAGE_META["description"] = "";
        $PAGE_META["keywords"] = "";

        $BREADCRUMB[] = array("url" => ENTRADA_RELATIVE . "/admin/sandbox", "title" => "Admin Side: Sandbox");

        $HEAD[]   = '<link rel="stylesheet" href="'.ENTRADA_URL.'/css/sandbox.css">';
        $JQUERY[] = '<script src="https://unpkg.com/vue@2.2.1"></script>';
        $JQUERY[] = '<script src="https://unpkg.com/axios@0.15.3/dist/axios.min.js"></script>';

At the beginning of every module file, we include checks for if the user can perform certain tasks (ie. `!$ENTRADA_ACL->amIAllowed("sandbox", "update")`), as well as the page meta and any CSS and Javascript files we want to include.

In this case, the two JS libraries we are including are VueJS 2 and Axios.

More info: 

- [VueJS 2](https://vuejs.org/v2/guide/)
- [Axios](https://github.com/mzabriskie/axios)

**Sidebar View Helper**

    $sidebar = new Views_Sandbox_Sidebar();
    $sidebar->render();

Here, we are including and immediately rendering a View helper. See section 7 below on what these are and how to create one yourself.

In this case, the sidebar renders a few links within the sidebar to easily access both the public and admin frontends of the Sandbox module.

**VueJS Hook Element**

    <div id="module-sandbox">

Every VueJS instance requires it be given a single element to watch and hook into. In this case, we will be hooking into the #module-sandbox element.

**Pagination Header**

    <div class="pagination clearfix">
        <button type="button" data-toggle="modal" data-target="#create-sandbox" class="btn btn-primary pull-left" v-if="current_user_can.create">
          <?php echo $translate->_("Create New Sandbox"); ?>
        </button>

        <p class="text-center summary">
          <?php echo $translate->_("Page {{ sandboxes.current_page }} / {{ sandboxes.last_page }} &mdash; {{ sandboxes.total }} Sandboxes Found"); ?>
        </p>

        <ul class="pull-right">
          <li v-bind:class="{ disabled:!sandboxes.prev_page_url }">
            <a href="#" aria-label="Previous" title="Previous" v-on:click.prevent="sandboxes.prev_page_url ? getSandboxes(sandboxes.prev_page_url) : null">
              <span aria-hidden="true">«</span>
            </a>
          </li>
          <li v-bind:class="{ disabled:!sandboxes.next_page_url }">
            <a href="#" aria-label="Next" v-on:click.prevent="sandboxes.next_page_url ? getSandboxes(sandboxes.next_page_url) : null">
              <span aria-hidden="true">»</span>
            </a>
          </li>
        </ul>
      </div>

There are a number of things going on in this code sample: 

The button to create a new sandbox will trigger opening the modal with an id of `create-sandbox`. Not only this, but it will also only display if the VueJS data setting `current_user_can.create` is set to `true`.

You will also notice a lot of curly braces around variable names. VueJS uses these to display the result of variables stored in memory. 

`v-on:click=methodName` will run a method upon the click event, while `v-bind:class` causes a classname to appear if a variable follows the logic given (ex. returns true).

`v-on:click.prevent` is a simple convenience shortform for `e.preventDefault()`. It can still be included within the method itself if you prefer.

More info: 

- [v-on](https://vuejs.org/v2/api/#v-on)
- [v-bind](https://vuejs.org/v2/api/#v-bind)
- [v-on modifiers (click.prevent)](https://vuejs.org/v2/api/#v-on)

**Table of Results**

    <div class="table-responsive">
        <table class="table table-borderless">
          <tr>
            <th width="150"><?php echo $translate->_("Title"); ?></th>
            <th width="200"><?php echo $translate->_("Description"); ?></th>
            <th><?php echo $translate->_("Created By"); ?></th>
            <th><?php echo $translate->_("Updated By"); ?></th>
            <th><?php echo $translate->_("Actions"); ?></th>
          </tr>
          <tr v-for="sandbox in sandboxes.data">
            <td>{{ sandbox.title }}</td>
            <td>{{ sandbox.description }}</td>
            <td><span v-if="sandbox.created_by">{{ sandbox.created_by.firstname }} {{ sandbox.created_by.lastname }}</span></td>
            <td><span v-if="sandbox.updated_by">{{ sandbox.updated_by.firstname }} {{ sandbox.updated_by.lastname }}</span></td>
            <td>
              <button class="edit-modal btn btn-warning" v-if="current_user_can.update" v-on:click.prevent="editSandbox(sandbox)">
                <span class="glyphicon glyphicon-edit"></span> <?php echo $translate->_("Edit"); ?>
              </button>
              <button class="edit-modal btn btn-danger" v-if="current_user_can.delete" v-on:click.prevent="deleteSandbox(sandbox.id)">
                <span class="glyphicon glyphicon-trash"></span> <?php echo $translate->_("Delete"); ?>
              </button>
            </td>
          </tr>
        </table>
      </div>

Here we want to display the list of sandboxes retrieved from the API.

`v-for="sandbox in sandboxes.data"` is an iterator over each item within `sandboxes.data`, and repeats the element it is attached to. 

More info: 

- [v-for](https://vuejs.org/v2/guide/list.html#v-for)

**Create and Edit Modals**

      <?php

      // Create sandbox modal
      $create_sandbox_modal = new Views_Gradebook_Modal([
        "id" => "create-sandbox",
        "title" => $translate->_("Create New Sandbox"),
      ]);

      $create_sandbox_modal->setBody('
        <form class="form-horizontal" v-on:submit.prevent="createSandbox">
          <div class="control-group" v-bind:class="{ error: form_errors.title }">
            <label class="control-label" for="newTitle">'.$translate->_("Title").'</label>
            <div class="controls">
              <input type="text" id="newTitle" placeholder="'.$translate->_("Title").'" v-model="new_sandbox.title">
              <p v-if="form_errors.title" class="error text-error">
                {{ form_errors.title }}
              </p>
            </div>
          </div>
          <div class="control-group" v-bind:class="{ error: form_errors.description }">
            <label class="control-label" for="newDescription">'.$translate->_("Description").'</label>
            <div class="controls">
              <textarea id="newDescription" placeholder="'.$translate->_("Description").'" v-model="new_sandbox.description"></textarea>
              <p v-if="form_errors.description" class="error text-error">
                {{ form_errors.description }}
              </p>
            </div>
          </div>

          <div class="control-group">
            <button type="submit" class="btn btn-success pull-right">'.$translate->_("Create Sandbox").'</button>
          </div>
        </form>
      ');

      $create_sandbox_modal->render();

      // Edit sandbox modal
      $edit_sandbox_modal = new Views_Gradebook_Modal([
        "id" => "edit-sandbox",
        "title" => $translate->_("Edit")." ".'"{{ update_sandbox.title }}"',
      ]);

      $edit_sandbox_modal->setBody('
        <form class="form-horizontal" v-on:submit.prevent="updateSandbox(update_sandbox.id)">
          <div class="control-group" v-bind:class="{ error: form_errors.title }">
            <label class="control-label" for="updateTitle">'.$translate->_("Title").'</label>
            <div class="controls">
              <input type="text" id="updateTitle" placeholder="'.$translate->_("Title").'" v-model="update_sandbox.title">
              <p v-if="form_errors.title" class="error text-error">
                {{ form_errors.title }}
              </p>
            </div>
          </div>
          <div class="control-group" v-bind:class="{ error: form_errors.description }">
            <label class="control-label" for="updateDescription">'.$translate->_("Description").'</label>
            <div class="controls">
              <textarea id="updateDescription" placeholder="'.$translate->_("Description").'" v-model="update_sandbox.description"></textarea>
              <p v-if="form_errors.description" class="error text-error">
                {{ form_errors.description }}
              </p>
            </div>
          </div>

          <div class="control-group">
            <button type="submit" class="btn btn-success pull-right">'.$translate->_("Update Sandbox").'</button>
          </div>
        </form>
      ');

      $edit_sandbox_modal->render();

      ?>

Here, we want to add the modals that will be used to create and edit sandboxes. 

Notice that we are using View helpers to do most of the heavy lifting -- in this case, the existing `Views_Gradebook_Modal` class came in handy.

**Including the custom JS file**

    </div>
        <?php

        // sandbox.js needs to run after DOM content
        echo '<script src="'.ENTRADA_URL.'/javascript/sandbox.js?release='.html_encode(APPLICATION_VERSION).'"></script>';
    }

Finally, we want to include the sandbox.js file that will contain our custom VueJS code. The reason why we include it last is because it needs to be run after the DOM content has been loaded.

#### 5. Create the new Public Module

Within `www-root/core/modules/public/sandbox.inc.php` place the following content:

    <?php
    /**
     * Entrada [ http://www.entrada-project.org ]
     *
     * @author Matt Simpson <simpson@queensu.ca>
     * @copyright Copyright 2016 Queen's University. All Rights Reserved.
     */

    if (!defined("PARENT_INCLUDED")) {
        exit;
    } elseif (!isset($_SESSION["isAuthorized"]) || !(bool) $_SESSION["isAuthorized"]) {
        header("Location: " . ENTRADA_URL);
        exit;
    } elseif (!$ENTRADA_ACL->amIAllowed("sandbox", "read")) {
        add_error("Your account does not have the permissions required to use this module.<br /><br />If you believe you are receiving this message in error please contact <a href=\"mailto:" . html_encode($AGENT_CONTACTS["administrator"]["email"]) . "\">" . html_encode($AGENT_CONTACTS["administrator"]["name"]) . "</a> for assistance.");

        echo display_error();

        application_log("error", "Group [" . $ENTRADA_USER->getActiveGroup() . "] and role [" . $ENTRADA_USER->getActiveRole() . "] do not have access to this module [" . $MODULE . "]");
    } else {
        $PAGE_META["title"] = "Public Side: Sandbox";
        $PAGE_META["description"] = "";
        $PAGE_META["keywords"] = "";

        $BREADCRUMB[] = array("url" => ENTRADA_RELATIVE . "/sandbox", "title" => "Public Side: Sandbox");

        $HEAD[]   = '<link rel="stylesheet" href="'.ENTRADA_URL.'/css/sandbox.css">';
        $JQUERY[] = '<script src="https://unpkg.com/vue@2.2.1"></script>';
        $JQUERY[] = '<script src="https://unpkg.com/axios@0.15.3/dist/axios.min.js"></script>';

        $sidebar = new Views_Sandbox_Sidebar();
        $sidebar->render();
        ?>
        <div id="module-sandbox">

          <div class="pagination clearfix">

            <p class="text-center summary">
              <?php echo $translate->_("Page {{ sandboxes.current_page }} / {{ sandboxes.last_page }} &mdash; {{ sandboxes.total }} Sandboxes Found"); ?>
            </p>

            <ul class="pull-right">
              <li v-bind:class="{ disabled:!sandboxes.prev_page_url }">
                <a href="#" aria-label="Previous" title="Previous" v-on:click.prevent="sandboxes.prev_page_url ? getSandboxes(sandboxes.prev_page_url) : null">
                  <span aria-hidden="true">«</span>
                </a>
              </li>
              <li v-bind:class="{ disabled:!sandboxes.next_page_url }">
                <a href="#" aria-label="Next" v-on:click.prevent="sandboxes.next_page_url ? getSandboxes(sandboxes.next_page_url) : null">
                  <span aria-hidden="true">»</span>
                </a>
              </li>
            </ul>
          </div>

          <div class="table-responsive">
            <table class="table table-borderless">
              <tr>
                <th width="150"><?php echo $translate->_("Title"); ?></th>
                <th width="200"><?php echo $translate->_("Description"); ?></th>
                <th><?php echo $translate->_("Created By"); ?></th>
                <th><?php echo $translate->_("Updated By"); ?></th>
              </tr>
              <tr v-for="sandbox in sandboxes.data">
                <td>{{ sandbox.title }}</td>
                <td>{{ sandbox.description }}</td>
                <td><span v-if="sandbox.created_by">{{ sandbox.created_by.firstname }} {{ sandbox.created_by.lastname }}</span></td>
                <td><span v-if="sandbox.updated_by">{{ sandbox.updated_by.firstname }} {{ sandbox.updated_by.lastname }}</span></td>
              </tr>
            </table>
          </div>

          <?php

          // Create sandbox modal
          $create_sandbox_modal = new Views_Gradebook_Modal([
            "id" => "create-sandbox",
            "title" => $translate->_("Create New Sandbox"),
          ]);

          $create_sandbox_modal->setBody('
            <form class="form-horizontal" v-on:submit.prevent="createSandbox">
              <div class="control-group" v-bind:class="{ error: form_errors.title }">
                <label class="control-label" for="newTitle">'.$translate->_("Title").'</label>
                <div class="controls">
                  <input type="text" id="newTitle" placeholder="'.$translate->_("Title").'" v-model="new_sandbox.title">
                  <p v-if="form_errors.title" class="error text-error">
                    {{ form_errors.title }}
                  </p>
                </div>
              </div>
              <div class="control-group" v-bind:class="{ error: form_errors.description }">
                <label class="control-label" for="newDescription">'.$translate->_("Description").'</label>
                <div class="controls">
                  <textarea id="newDescription" placeholder="'.$translate->_("Description").'" v-model="new_sandbox.description"></textarea>
                  <p v-if="form_errors.description" class="error text-error">
                    {{ form_errors.description }}
                  </p>
                </div>
              </div>

              <div class="control-group">
                <button type="submit" class="btn btn-success pull-right">'.$translate->_("Create Sandbox").'</button>
              </div>
            </form>
          ');

          $create_sandbox_modal->render();

          // Edit sandbox modal
          $edit_sandbox_modal = new Views_Gradebook_Modal([
            "id" => "edit-sandbox",
            "title" => $translate->_("Edit")." ".'"{{ update_sandbox.title }}"',
          ]);

          $edit_sandbox_modal->setBody('
            <form class="form-horizontal" v-on:submit.prevent="updateSandbox(update_sandbox.id)">
              <div class="control-group" v-bind:class="{ error: form_errors.title }">
                <label class="control-label" for="updateTitle">'.$translate->_("Title").'</label>
                <div class="controls">
                  <input type="text" id="updateTitle" placeholder="'.$translate->_("Title").'" v-model="update_sandbox.title">
                  <p v-if="form_errors.title" class="error text-error">
                    {{ form_errors.title }}
                  </p>
                </div>
              </div>
              <div class="control-group" v-bind:class="{ error: form_errors.description }">
                <label class="control-label" for="updateDescription">'.$translate->_("Description").'</label>
                <div class="controls">
                  <textarea id="updateDescription" placeholder="'.$translate->_("Description").'" v-model="update_sandbox.description"></textarea>
                  <p v-if="form_errors.description" class="error text-error">
                    {{ form_errors.description }}
                  </p>
                </div>
              </div>

              <div class="control-group">
                <button type="submit" class="btn btn-success pull-right">'.$translate->_("Update Sandbox").'</button>
              </div>
            </form>
          ');

          $edit_sandbox_modal->render();

          ?>
            
        </div>
        <?php

        // sandbox.js needs to run after DOM content
        echo '<script src="'.ENTRADA_URL.'/javascript/sandbox.js?release='.html_encode(APPLICATION_VERSION).'"></script>';
    }

It is identical to the admin version, except we have removed the create, edit and delete buttons, for a read-only experience.

#### 6. Integrate VueJS and Axios for a single-page app experience

In order for us to take advantage of the features of VueJS, we need to create a custom javascript file. 

Create a file at `www-root/javascript/sandbox.js`, then insert the following content:
    
    // Set defaults
    axios.defaults.baseURL = API_URL + '/sandbox'
    axios.defaults.headers.common["Authorization"] = 'Bearer ' + JWT

    // Intercept response
    axios.interceptors.response.use(function(response) {

      // If response has a new authorization header, replace it as default JWT
      if (response.headers.authorization) {
        axios.defaults.headers.common["Authorization"] = response.headers.authorization
      }

      return response
    })

    // Init Vue instance
    var vm = new Vue({
      el: '#module-sandbox',

      data: {
        sandboxes: {
          total: 0,
          per_page: 0,
          current_page: 1,
          last_page: 0,
          next_page_url: null,
          prev_page_url: null,
          from: 0,
          to: 0,
          data: []
        },
        current_user_can: {
          read: false,
          create: false,
          update: false,
          delete: false
        },
        'new_sandbox': {
          title: '',
          description: ''
        },
        'update_sandbox': {
          id: 0,
          title: '',
          description: ''
        },
        form_errors: {
          title: '',
          description: ''
        }
      },  

      mounted: function() {
        this.getSandboxes()
      },

      watch: {
        
        // When entering new data, empty out form errors

        'new_sandbox.title': function (newValue) {
          this.form_errors.title = ''
        },

        'new_sandbox.description': function (newValue) {
          this.form_errors.description = ''
        },

        'update_sandbox.title': function (newValue) {
          this.form_errors.title = ''
        },

        'update_sandbox.description': function (newValue) {
          this.form_errors.description = ''
        },

      },

      methods: {
        
        getSandboxes: function(url) {
          var url = url ? url : axios.defaults.baseURL

          axios.get(url).then((response) => {
            this.sandboxes = Object.assign({}, response.data.sandboxes)
            this.current_user_can = Object.assign({}, response.data.current_user_can)
          })
        },

        getPage: function(page) {
          this.getSandboxes(axios.defaults.baseURL + '?page=' + page)
        },

        createSandbox: function() {
          var params = new URLSearchParams();
          params.append('title', this.new_sandbox.title)
          params.append('description', this.new_sandbox.description)

          axios.post('', params).then((response) => {
            this.getPage(this.sandboxes.current_page)

            this.new_sandbox = {
              title: '',
              description: ''
            }

            jQuery('#create-sandbox').modal('hide')
          })
          .catch(this.handleFormErrors)
        },

        editSandbox: function(sandbox) {
          this.update_sandbox.id = sandbox.id
          this.update_sandbox.title = sandbox.title
          this.update_sandbox.description = sandbox.description

          this.form_errors = {
            title: '',
            description: ''
          }

          jQuery("#edit-sandbox").modal('show');
        },

        updateSandbox: function(id) {
          var params = new URLSearchParams();
          params.append('title', this.update_sandbox.title)
          params.append('description', this.update_sandbox.description)

          axios.put('/' + id, params).then((response) => {
            this.getPage(this.sandboxes.current_page)

            this.update_sandbox = {
              id: 0,
              title: '',
              description: ''
            }

            jQuery("#edit-sandbox").modal('hide');
          })
          .catch(this.handleFormErrors)
        },

        deleteSandbox: function(id) {
          if (confirm('Are you sure you want to delete this sandbox?')) {
            axios.delete('/' + id).then((response) => {
              this.getPage(this.sandboxes.current_page)
            })
          }
        },

        handleFormErrors: function(error) {
          if (error.response && error.response.data.title) {
            this.form_errors.title = error.response.data.title[0]
          }

          if (error.response && error.response.data.description) {
            this.form_errors.description = error.response.data.description[0]
          }
        }
      },
    })

Let's go through this step by step: 

**Setting up common defaults and automatic token renewal**

    // Set defaults
    axios.defaults.baseURL = API_URL + '/sandbox'
    axios.defaults.headers.common["Authorization"] = 'Bearer ' + JWT

    // Intercept response
    axios.interceptors.response.use(function(response) {

      // If response has a new authorization header, replace it as default JWT
      if (response.headers.authorization) {
        axios.defaults.headers.common["Authorization"] = response.headers.authorization
      }

      return response
    })

First, you will want to use the global JS variables API_URL and JWT to set up the default values.

Next, we want to make sure we handle automatic token renewals coming from the `jwt.refreshWhenExpires` middleware on the API side. 

Since the new token is always attached to the response, we use a response interceptor to automatically reset the default Authorization header.

**Initializing the VueJS instance**

To initialize the VueJS instance, we need to set a few different keys: 

- `el`: the element the instance binds to

- `data`: the default state of the data being stored in memory

- `mounted`: sort of like document.ready in the jQuery world, this is a function that runs upon starting up the instance.

- `watches`: these are event-based functions that run whenever the variable gets changed. 

- `methods`: where all the logic lives for performing CRUD operations.

#### 7. Create the new View Helpers

A view helper allows you to create a single block of reusable code that can be displayed in many places. There are quite a few presently undocumented options and features of our view helpers so we would encourage you to review the files within the `www-root/core/library/Views/` directory for more information. Developer Tip: `HTMLTemplate.php` is an especially interesting feature.

You will notice that if you visit [http://entrada-1x-me.dev/sandbox](http://entrada-1x-me.dev/sandbox) in your web browser that you will now see a PHP error. That is due to the fact that we are using **view helpers** on lines 30 - 31 of `www-root/core/modules/public/sandbox.inc.php` but have not yet created the files.

Proceed with creating the two new files to accommodate the `Views_Sandbox_Sidebar` and `Views_Sandbox_Form` view helpers:

**Sandbox/Form.php** (1 of 2)

Within `www-root/core/library/Views/Sandbox/Form.php` place the following content:

    <?php
    /**
     * Entrada [ http://www.entrada-project.org ]
     */
    
    class Views_Sandbox_Form extends Views_HTML {
    
        protected function validateOptions($options = array()) {
            return $this->validateIsSet($options, array("action_url", "cancel_url", "title", "description"));
        }
    
        protected function renderView($options = array()) {
            global $translate;
    
            /*
             * $options["action_url"] is specified as a required in the validateOptions() method
             * defined above. We can safely use it here.
             */
            $action_url = $options["action_url"];
            $cancel_url = $options["cancel_url"];
    
            $title = $options["title"];
            $description = $options["description"];
            ?>
            <form class="form-horizontal" action="<?php echo $action_url ?>" method="POST">
                <input type="hidden" name="step" value="2" />
                <div class="control-group">
                    <label class="control-label form-required" for="sandbox-title"><?php echo $translate->_("Sandbox Title"); ?></label>
                    <div class="controls">
                        <input type="text" class="input-xxlarge" name="title" id="sandbox-title" value="<?php echo html_encode($title); ?>" />
                    </div>
                </div>
                <div class="control-group">
                    <label class="control-label form-nrequired" for="sandbox-description"><?php echo $translate->_("Sandbox Description"); ?></label>
                    <div class="controls">
                        <textarea class="input-xxlarge expandable" name="description" id="sandbox-description"><?php echo html_encode($description); ?></textarea>
                    </div>
                </div>
                <div class="row-fluid">
                    <a href="<?php echo $cancel_url; ?>" class="btn btn-default pull-left"><?php echo $translate->_("Cancel"); ?></a>
                    <input type="submit" class="btn btn-primary pull-right" value="<?php echo $translate->_("Submit"); ?>" />
                </div>
            </form>
            <?php
        }
    }


**Sandbox/Sidebar.php** (2 of 2)

Within `www-root/core/library/Views/Sandbox/Sidebar.php` place the following content:

    <?php
    /**
     * Entrada [ http://www.entrada-project.org ]
     */
    
    class Views_Sandbox_Sidebar extends Views_HTML {
        /**
         * Render the sidebar target.
         *
         * @param $options
         */
        protected function renderView($options = array()) {
            global $translate, $ENTRADA_ACL;
    
            $sidebar_html  = "<ul class=\"nav nav-list\">";
            $sidebar_html .= "    <li><a href=\"" . ENTRADA_RELATIVE . "/sandbox\">" . $translate->_("Public Side") . "</a></li>";
    
            if ($ENTRADA_ACL->amIAllowed("sandbox", "create", false)) {
                $sidebar_html .= "<li><a href=\"" . ENTRADA_RELATIVE . "/admin/sandbox\">" . $translate->_("Admin Side") . "</a></li>";
            }
    
            $sidebar_html .= "</ul>";
    
            new_sidebar_item($translate->_("Sandbox Sidebar"), $sidebar_html, "page-sandbox", "open");
        }
    }

### Finishing Tasks

At this point you should have a relatively functional module that allows you to create, read, update, and delete items from the `entrada.sandbox` table. There a few final tasks to complete the new module.
 
#### 1. Add an ACL Array Entry
 
Entrada's ACL is quite powerful, albeit slightly complex. For more information on exactly how the Access Control Level feature works please visit the [Entrada ACL](developer/entrada-acl) section.

Within the `www-root/core/library/Entrada/authentication/entrada_acl.inc.php` file you will want to add the `resource_type` that you created back in the migration to the `$modules` array. This will allow groups other than `medtech:admin` to access your module based on `entrada_auth.acl_permissions` records that have been setup for this `resource_type`.
 
For example:

	var $modules = array (
		"mom" => array (
            ...
            "reportindex",
            "sandbox",
            "quiz" => array (
                "quizquestion",
                "quizresult"
            ),
            ...
    );

#### 2. Add an Admin Menu Entry

The **Admin** tab within the primary navigation near the top of the Entrada user interface would benefit from having a "Manage Sandbox" link added to it.
 
In order to do this add the following line to the `$MODULES` section of the `www-root/core/config/settings.inc.php` file:

    $MODULES["sandbox"] = array("title" => "Manage Sandbox", "resource" => "sandbox", "permission" => "create");
    
This will give anyone with _create_ access to the sandbox resource a menu item within the Admin tab. 

## Getting Help

This Quickstart Guide is intended to give you an overview of how to create a new module within Entrada ME 1.8. It will also give you a glimpse into how Entrada operates. If you would like a more indepth one-on-one tutorial, please contact the Entrada Consortium team or reach out in our Slack channel.
