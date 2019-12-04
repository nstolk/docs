# **Moodle Advanced Block Tutorial**

## Recap

Currently we have the following files:

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
            |-- edit_form.php
            |
            |-- settings.php
            |
            |-- version.php
            
The file contents are as follows:

### db/access.php

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
    
### lang/en/block_nicksblock.php

    <?php
    
    $string['pluginname'] = 'Nick\'s Basic block';
    $string['nicksblock'] = 'Nick\'s Block Settings'; // Default title for block
    $string['nicksblock:addinstance'] = 'Add a new HTML block';
    $string['nicksblock:myaddinstance'] = 'Add a new HTML block to the My Moodle page';
    $string['blockstring'] = 'Default value for Blockstring as defined in the language file';
    $string['blocktitle'] = 'Default title for this block';
    $string['headerconfig'] = 'Nick\'s Block Configuration';
    $string['descconfig'] = 'HTML settings';
    $string['labelallowhtml'] = 'Allow HTML?';
    $string['descallowhtml'] = 'Checking this option will strip HTML tags';
    
 ### block_nicksblock.php
 
     <?php
     
     class block_nicksblock extends block_base {
         public function init() {
             $this->title = get_string('nicksblock', 'block_nicksblock');
         }
     
         public function get_content() {
             if ($this->content !== null) {
                 return $this->content;
             }
     
             $this->content = new stdClass;
             $this->content->text = get_string('blockstring', 'block_nicksblock');
             $this->content->footer = 'Block footer';
     
             return $this->content;
         }
     
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
     
         public function instance_allow_multiple() {
             return true;
         }
     
         // This function is required to enable global configuration
         public function has_config() {
             return true;
         }
     
         public function instance_config_save($data, $nolongerused = false) {
             if(get_config('nicksblock', 'Allow_HTML') == '0') {
                 $data->text = strip_tags($data->text);
             }
     
             return parent::instance_config_save($data, $nolongerused);
         }
     
         public function hide_header() {
             return true;
         }
     
         public function html_attributes() {
             $attributes = parent::html_attributes(); // Get default values
             $attributes['class'] .= ' block_' . $this->name(); // Append our class to class attribute
             return $attributes;
         }
     
         public function applicable_formats() {
             return array('site-index' => true);
         }
     }
     
### edit_form.php

    <?php
    
    class block_nicksblock_edit_form extends block_edit_form {
    
        protected function specific_definition($mform) {
            // Section header title according to language file.
            $mform->addElement('header', 'config_header', get_string('blocksettings', 'block'));
    
            // A sample string variable with a default value.
            $mform->addElement('text', 'config_text', get_string('blockstring', 'block_nicksblock'));
            $mform->setDefault('config_text', 'default value');
    
            $mform->setType('config_text', PARAM_RAW);
    
            $mform->addElement('text', 'config_title', get_string('blocktitle', 'block_nicksblock'));
            $mform->setDefault('config_title', 'default value');
            $mform->setType('config_title', PARAM_TEXT);
        }
    }
    
### settings.php

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

### version.php 

    <?php
    
    $plugin->component = 'block_nicksblock';
    $plugin->version = 2019112000;
    $plugin->requires = 2019111800;
    
## Adding forms

Create the file **nicksblock_form.php**

    <?php
    
    require_once("{$CFG->libdir}/formslib.php");
    
    class nicksblock_form extends moodleform {
    
        function definition() {
            // Quick form
            $mform =& $this->_form;
            $mform->addElement('header', 'displayinfo', get_string('textfields', 'block_nicksblock'));
        }
    }
    
Create the file **view.php**

    <?php
    
    require_once('../../config.php');
    require_once('nicksblock_form.php');
    
    global $DB;
    
    $courseid = required_param('courseid', PARAM_INT);
    
    if (!$course = $DB->get_record('course', array('id' => $courseid))) {
        print_error('invalidcourse', 'block_nicksblock', $courseid);
    }
    
    require_login($course);
    
    $nicksblock = new nicksblock_form();
    
    $nicksblock->display();
    
In block_nicksblock.php - replace the contents of the method get_content() with this

    public function get_content() {
            global $COURSE;
    
            if ($this->content !== null) {
                return $this->content;
            }
    
            $this->content = new stdClass;
            $this->content->text = get_string('blockstring', 'block_nicksblock');
    
            $url = new moodle_url('/blocks/nicksblock/view.php', array('blockid' => $this->instance->id, 'courseid' => $COURSE->id));
            $this->content->footer = html_writer::link($url, get_string('addpage', 'block_nicksblock'));
    
            return $this->content;
        }
        
Add these lines to your language file

    $string['addpage'] = 'Add a new page';
    $string['textfields'] = 'Text fields';
    
Now when you go to your Moodle site, nicksblock will look like this:

![](img/advanced_block/moodle_add_a_new_page.png)

the URL will contain the view.php line plus some additional query strings

    view.php?blockid=21&courseid=1
    
Block ID will refer to the instance of our block.
Let's find this number in our database.
![](img/advanced_block/moodle_block_instance_table.png)

In the table mdl_block_instances you'll see that ID 21 corresponds with an instance of nicksblock that's being displayed on the admin-search page.

If we try to access view.php right now we'll see this

![](img/advanced_block/moodle_view_php.png)

Right now it's just a basic form. This is because view.php is being accessed directly by the URL and not by being included in another page.

##Add a Header 

Replace the contents of the view.php with this

    <?php
    
    require_once('../../config.php');
    require_once('nicksblock_form.php');
    
    global $DB, $OUTPUT, $PAGE;
    
    $courseid = required_param('courseid', PARAM_INT);
    $blockid = required_param('blockid', PARAM_INT);
    $id = optional_param('id', 0, PARAM_INT);
    
    if (!$course = $DB->get_record('course', array('id' => $courseid))) {
        print_error('invalidcourse', 'block_nicksblock', $courseid);
    }
    
    require_login($course);
    
    $PAGE->set_url('/blocks/nicksblock/view.php', array('id' => $courseid));
    $PAGE->set_pagelayout('standard');
    $PAGE->set_heading(get_string('edithtml', 'block_nicksblock'));
    
    $settingsnode = $PAGE->settingsnav->add(get_string('nicksblock', 'block_nicksblock'));
    $editurl = new moodle_url('/blocks/nicksblock/view.php', array('id' => $id, 'courseid' => $courseid, 'blockid' => $blockid));
    $editnode = $settingsnode->add(get_string('editpage', 'block_nicksblock'), $editurl);
    $editnode->make_active();
    
    $nicksblock = new nicksblock_form();
    
    echo $OUTPUT->header();
    $nicksblock->display();
    echo $OUTPUT->footer();
    
view.php will now have a header and the standard Moodle layout.
Note here that  

    $PAGE->settingsnav
    
applies to the settings block and not the navigation block.

The layout of the view.php will look like this
    
![](img/advanced_block/moodle_view_php_header.png)

If you click on 'Edit page', you'll see the query strings have changed, namely it has added an id query string.

        view.php?id=0&courseid=1&blockid=21

Currently the ID query string doesn't do anything **(?)**
