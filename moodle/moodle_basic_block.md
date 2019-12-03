# **Moodle Block Tutorial**

    blocks/nicksblock
        |
        |-- db/
        |    |
        |    |-- access.php
        |
        |-- lang/en
        |    |
        |    |-- block_nicksblock.php
        |
        |-- block_nicksblock.php
        |
        |-- version.php

## block_nicksblock.php

    <?php
    class block_nicksblock extends block_base {
        public function init() {
            $this->title = get_string('nicksblock', 'block_nicksblock');
        }
    }

## /db/access.php

    <?php
        $capabilities = array(

        'block/nicksblock:myaddinstance' => array(
            'captype' => 'write',
            'contextlevel' => CONTEXT_SYSTEM,
            'archetypes' => array(
                'user' => CAP_ALLOW
            ),

            'clonepermissionsfrom' => 'moodle/my:manageblocks'
        ),

        'block/nicksblock:addinstance' => array(
            'riskbitmask' => RISK_SPAM | RISK_XSS,

            'captype' => 'write',
            'contextlevel' => CONTEXT_BLOCK,
            'archetypes' => array(
                'editingteacher' => CAP_ALLOW,
                'manager' => CAP_ALLOW
            ),

            'clonepermissionsfrom' => 'moodle/site:manageblocks'
        ),
    );

## /lang/en/block_nicksblock.php

    <?php

    $string['pluginname'] = 'Nick\'s HTML block';
    $string['nicksblock'] = 'Nick\'s Block';
    $string['nicksblock:addinstance'] = 'Add a new Nick HTML block';
    $string['nicksblock:myaddinstance'] = 'Add a new Nick HTML block to the My Moodle page';

## version.php

    <?php

    $plugin->component = 'block_nicksblock';  // Recommended since 2.0.2 (MDL-26035). Required since 3.0 (MDL-48494)
    $plugin->version = 2019112100;  // YYYYMMDDHH (year, month, day, 24-hr time)
    $plugin->requires = 2010112500; // YYYYMMDDHH (This is the release version for Moodle 2.0)

## Plugin check

When you reload your Moodle site, it'll redirect to the plugin check page.

![Plugin check](img/basic_block/moodle_plugins_check.png)

Update the database, your plugin should now be added to your Moodle site.

![](img/basic_block/moodle_adding_plugin.png)


## Add your block

You can now add your block by turning blocks editing on (click the gear icon or the button)
Then click 'Add a block', and select Nick's HTML block.

![](img/basic_block/moodle_add_block.png)

The block should now appear on the page.

![](img/basic_block/moodle_nicks_block.png)

## Block content

    <?php

    class block_nicksblock extends block_base {
        public function init() {
            $this->title = get_string('nicksblock', 'block_nicksblock');
        }

        public function get_content() {
            if ($this->content !== null) {
                return $this->content;
            }

            $this->content         =  new stdClass;
            $this->content->text   = 'The content of Nick\'s block.';
            $this->content->footer = 'Footer here...';

            return $this->content;
        }
    }

![](img/basic_block/moodle_block_content.png)

## Instance configuration

Create a new file: edit_form.php

    <?php

    class block_nicksblock_edit_form extends block_edit_form {

        protected function specific_definition($mform) {

            // Section header title according to language file.
            $mform->addElement('header', 'config_header', get_string('blocksettings', 'block'));

            // A sample string variable with a default value.
            $mform->addElement('text', 'config_text', get_string('blockstring', 'block_nicksblock'));
            $mform->setDefault('config_text', 'default value');
            $mform->setType('config_text', PARAM_RAW);

        }
    }

Add this line to the language file:

    $string['blockstring'] = 'Sample string variable';

You may need to clear your site's cache for this to show up.
(Site Administration -> Development -> Purge all caches)

![](img/basic_block/moodle_example_string.png)

## Customize the title of the block

Add this to edit_form.php

    $mform->addElement('text', 'config_title', get_string('blocktitle', 'block_nicksblock'));
            $mform->setDefault('config_title', 'default value');
            $mform->setType('config_title', PARAM_TEXT);

Add this block_nicksblock.php

    public function specialization() {
            if (isset($this->config)) {
                if (empty($this->config->title)) {
                    $this->title = get_string('defaulttitle', 'block_nicksblock');
                } else {
                    $this->title = $this->config->title;
                }

                if (empty($this->config->text)) {
                    $this->config->text = get_string('defaulttext', 'block_nicksblock');
                }
            }
        }

Add this to the language file

    $string['blocktitle'] = 'Title for Nick\'s Block';


![](img/basic_block/moodle_change_block_title.png)

![](img/basic_block/moodle_block_title_changed.png)

## Multiple instances of the same block

Add this to block_nicksblock.php

    public function instance_allow_multiple() {
            return true; 
        }

## Global configuration

Create a new file called settings.php

    <?php

    $settings->add(new admin_setting_heading(
        'headerconfig',
        get_string('headerconfig', 'block_nicksblock'),
        get_string('descconfig', 'block_nicksblock')
    ));

    $settings->add(new admin_setting_configcheckbox(
        'nicksblock/Allow_HTML',
        get_string('labelallowhtml', 'block_nicksblock'),
        get_string('descallowhtml', 'block_nicksblock'),
        '0'
    ));

Add the following function to block_nicksblock.php

    function has_config() {
        return true;
    }
    
Add the following function to block_nicksblock.php

    public function instance_config_save($data, $nolongerused = false) {
        if(get_config('nicksblock', 'Allow_HTML') == '0') {
            $data->text = strip_tags($data->text);
        }
        
        return parent::instance_config_save($data, $nolongerused);
    }
    
Add these lines to your language file

    $string['headerconfig'] = 'Nick\'s Block Configuration';
    $string['descconfig'] = 'HTML settings';
    $string['labelallowhtml'] = 'Allow HTML?';
    $string['descallowhtml'] = 'Checking this option will strip HTML tags';
    
You'll find these settings (Site Administration -> Plugins -> Blocks -> Nick's Basic Block

![](img/basic_block/moodle_global_settings.png)

## Eye Candy

The following code will hide the block header

    public function hide_header() {
        return true;
    }

You can also add custom HTML attributes to your block.
The following code will add the class 'block_nicksblock' to the div element containing our block.

    public function instance_config_save($data, $nolongerused = false) {
        if(get_config('nicksblock', 'Allow_HTML') == '0') {
            $data->text = strip_tags($data->text);
        }

        return parent::instance_config_save($data, $nolongerused);
    }
    
Before:

![](img/basic_block/moodle_attributes_before.png)

After:

![](img/basic_block/moodle_attributes_after.png)
    
## Block restriction

You can restrict adding your block to certain pages only (site home, or admin page)
Here is such an example

    public function applicable_formats() {
        return array('site-index' => true);
    }

The above code will only allow you to add Nick's Block to the site home page.
