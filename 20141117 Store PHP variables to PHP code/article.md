# Store PHP variables to PHP code

One of those crazy ideas once crossed my mind has proven to be very successful. It has managed to get into many of my projects with no problems so far.

The idea of storing a PHP variable - any kind of variable - into a regular PHP file serves the same purpose as storing it in the form of JSON, XML or a serialized PHP string.

What is better with this method is mainly speed. The system reads native PHP code faster than parsing JSON, XML, PHP serialized strings etc. And there is more to it. The storage takes the form or a PHP file so it can't be accessed by a direct call from a web browser. It is readable, it is editable, and did I mention it's blazing fast?

So lets jump straight into the code. It is a relatively small script after all.

    /**
     * PhpVarEncoder
     *
     * @version 1.0
     * @author Creative Pulse
     * @copyright Creative Pulse 2014
     * @license http://www.gnu.org/licenses/gpl-2.0.html GNU/GPL
     * @link http://www.creativepulse.gr
     */

    class PhpVarEncoder {
        public $error_code;
        public $error_msg;
        public $output;
        public $padding = "\t";
        public $lf = "\r\n";
        private $level;
        public $esc_hi_bit = false;

        public function obj_init($inst_name, $service, $srv_group, $pars = null) {
            parent::obj_init($inst_name, $service, $srv_group, $pars);
            $this->is_default = true;
            $this->version = '2.0-dev2014';
        }

        public function esc_str($input) {
            $result = '';
            for ($i = 0, $len = strlen($input); $i < $len; $i++) {
                $c = $input[$i];
                if ($c == '"') {
                    $result .= '\\"';
                }
                else if ($c == '\\') {
                    $result .= '\\\\';
                }
                else if ($c == '$') {
                    $result .= '\\$';
                }
                else if ($c == "\n" || $c == "\r" || $c == "\t" || $c >= ' ') {
                    $result .= $c;
                }
                else if ($c < ' ' || ($this->esc_hi_bit && $c > '~')) {
                    $result .= sprintf('\\x%02X', ord($c));
                }
                else {
                    $result .= $c;
                }
            }
            return $result;
        }

        private function recurse_encode($data) {
            if ($this->level == 0) {
                $this->output .= '$__v = ';
            }

            $data_type = gettype($data);
            switch ($data_type) {
                case 'boolean':
                    $this->output .= ($data ? 'true' : 'false') . ',';
                    break;

                case 'integer':
                case 'double':
                    $this->output .= $data . ',';
                    break;

                case 'string':
                    $this->output .= '"' . $this->esc_str($data) . '",';
                    break;

                case 'NULL':
                    $this->output .= 'null,';
                    break;

                case 'object':
                    $data = get_object_vars($data);
                case 'array':
                    $plain_array = false;
                    if ($data_type == 'array') {
                        $plain_array = true;
                        $idx = -1;
                        foreach (array_keys($data) as $k) {
                            $idx++;
                            if ($idx !== $k) {
                                $plain_array = false;
                                break;
                            }
                        }
                    }

                    $this->level++;
                    $this->output .= 'array(' . $this->lf . str_repeat($this->padding, $this->level);

                    $first = true;
                    foreach ($data as $k => $v) {
                        if ($first) {
                            $first = false;
                        }
                        else {
                            $this->output .= $this->lf . str_repeat($this->padding, $this->level);
                        }

                        if (!$plain_array) {
                            switch (gettype($k)) {
                                case 'integer':
                                case 'double':
                                    $this->output .= $k . ' => ';
                                    break;

                                case 'string':
                                    $this->output .= '"' . $this->esc_str($k) . '" => ';
                                    break;

                                default:
                                    $this->error_code = 'UNSUPPORTED_DATA_TYPE';
                                    $this->error_msg = 'Attempted to encode to an unsupported data type';
                                    return;
                            }
                        }

                        $this->recurse_encode($v);
                        if ($this->error_code != '') {
                            break;
                        }
                    }

                    $this->level--;
                    $this->output .= $this->lf . str_repeat($this->padding, $this->level) . '),';

                    break;

                case 'resource':
                    $this->error_code = 'UNSUPPORTED_DATA_TYPE';
                    $this->error_msg = 'Attempted to encode to an unsupported data type';
                    break;

                default:
                    $this->error_code = 'UNRECOGNIZED_DATA_TYPE';
                    $this->error_msg = 'Attempted to encode to an unrecognized data type';
            }

            if ($this->level == 0) {
                $this->output[strlen($this->output) - 1] = ';';
            }
        }

        public function encode($data, $return_result = true) {
            $this->error_code = '';
            $this->error_msg = '';
            $this->output = '';
            $this->level = 0;

            $this->recurse_encode($data);

            if ($this->error_code != '') {
                return false;
            }
            else if ($return_result) {
                return $this->output;
            }
            else {
                return true;
            }
        }

        public function save($filename) {
            return @file_put_contents($filename, "<?php\n\n" . $this->output . "\n\n?>");
        }

        public function encode_save($filename, $data) {
            if (!$this->encode($data, false)) {
                return false;
            }
            return $this->save($filename);
        }
    }

That was the script you can copy/paste to your project. Before you start, let's see some examples. The input can be any form of a PHP variable. In this case let's take a very demanding array, one that contains many different types of variables including an object.

    $obj = new stdClass();
    $obj->property1 = 'value1';
    $obj->property2 = 'value2';

    $var = array(
        'key1' => 123,
        'key2' => 'abc' . "\t" . 'def' . "\0ghi",
        'key3' => array(
            'sub_key1' => false,
            'sub_key2' => null,
            7 => 'string value'
        ),
        'key4' => $obj
    );

There are a couple of ways you can use the script. In this example we will try to save this array straight into a file. All you have to do is create an instance of the class and call the encode_save() method:

    $encoder = new PhpVarEncoder();
    $encoder->encode_save('/path/to/file.php', $var);

The output file gets this content:

    $__v = array(
        "key1" => 123,
        "key2" => "abc  def\x00ghi",
        "key3" => array(
            "sub_key1" => false,
            "sub_key2" => null,
            7 => "string value",
        ),
        "key4" => array(
            "property1" => "value1",
            "property2" => "value2",
        ),
    );

You can now require() or include() this file from anywhere in your project and load the variable as `$__v` .

If you wonder why the script saves the content in such a weird named variable it is because this name ($__v) is short and at the same time very uncommon. That way there is minimal chance it conflicts other variables in your project.

This article was originally published in [http://www.creativepulse.gr/en/blog/2014/store-php-variables-to-php-code](http://www.creativepulse.gr/en/blog/2014/store-php-variables-to-php-code) on 2014-11-17.
