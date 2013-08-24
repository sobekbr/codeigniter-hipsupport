# HipSupport 

HipSupport is a CodeIgniter library that facilitates the creation of a live chat support system ontop of HipChat's API. If you are already using CodeIgniter and HipChat, then you can have a fully functional live chat system up and running in minutes.

- [How it Works](#how-it-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Quickstart](#quickstart)
- [Named Users](#named-users)
- [Limitations](#limitations)

## How it Works

HipChat released a [jQuery HipChat Plugin](http://blog.hipchat.com/2013/08/20/embedding-hipchat/) that allows users to embed a [HipChat Web Client](http://help.hipchat.com/knowledgebase/articles/238941-embedding-hipchat) into their web pages so that users can now access public chat rooms anonymously. HipSupport uses [HipChat's API](https://github.com/hipchat/hipchat-php) to dynamically create a new public chat room for each incoming chat request and send a notification to your company's users of this new chat session. From there, your company's users can join the room and chat with the user.

![HipChat Embedded Chat Client](http://www.bradestey.com/img/projects/hipsupport/hipchat-embed.png "HipChat Embedded Chat Client")

## Requirements

1. PHP 5.2+
2. CodeIgniter 2.0 - 2.1.4

## Installation

Place the `config/hipsupport.php` file in your `application/config` directory. Place the `libraries/HipSupport.php` and `libraries/HipChat.php` files into your `application/libraries` directory. The `libraries/HipChat.php` is a modified version of the official [HipChat PHP Library](https://github.com/hipchat/hipchat-php) modified to work with CodeIgniter. To load this library use:

    $this->load->library('hipsupport');

## Configuration

An Admin token is required in the `token` field and a user with access to add rooms is required in the `owner_user_id` field. 

The `room_name` format should be something unique and by default appends `Y-m-d H:m` to the end of the Room name. 

    'room_name' => 'Live Chat ' . date('Y-m-d H:m') 

Or you can leave this blank and assign it at runtime with whatever you want. Like the user's IP Address or whatever. If a room name already exists then a number will be appended to the end of the name. (This comes at the cost of an extra API request, so be mindful of that. See the <a href="#limitations">Limitations</a> section for more details.)

To send notifications, you must define the `room_id` in the `notification` array. If the `room_id` is `null` then no notification will be sent.

## Quickstart

To bring HipSupport online manually, use `$this->hipsupport->online()`. You can create controllers for these methods to call them from the command line. If HipSupport is Offline then `$this->hipsupport->init()` will return `false`.

To create an absolute basic installation, create a controller to handle your incoming chat requests. 

    class Chat extends CI_Controller {

      public function index()
      {
        $this->load->library('hipsupport');
        $room = $this->hipsupport->init();
        if ($room) redirect($room->hipsupport_url);
      }

    }

In your view, add a form to post into your chat route when HipSupport is online.

    if ($this->hipsupport->isOnline()) :
      echo form_open('email/send');
      echo form_submit('', ''Start Live Chat');
      echo form_close();
    endif; 

By using JavaScript, you can open the chat up on an iFrame, in an iFrame inside a modal, or open the chat in a new window. The chat screen is simple and automatically resizes to the size of its container. To handle an ajax request, your route would look something like this:


    class Chat extends CI_Controller {

      public function index()
      {
        $this->load->library('hipsupport');
        $room = $this->hipsupport->init();
        if ($room) 
        {
          if ($this->input->is_ajax_request())
          {
            echo json_encode(array('url' => $room->hipsupport_url));
            exit();
          }
          redirect($room->hipsupport_url);
        }
      }

    }

## Limitations

### Named Users

HipChat's API doesn't currently provide a way to name Guests. If you pass `$this->hipsupport->init(array('anonymous' => false))` then HipChat will prompt the user to enter there name, but the page is kind of clunky, isn't responsive and kills the illusion of a live chat system. Alternatively, you can add a form and a layer of validation in front of the `$this->hipsupport->init()` method and pass the user's inputs into the `room_name` and notification `message`. You can even save the user's data (User ID, Name, Email, etc) in your database associated to the chat's `room_id` so that you can attribute the chat history to specific users.

Here's an example of an extremely basic validation.

    class Chat extends CI_Controller {

      public function index()
      {
        $this->load->library('hipsupport');
        $this->load->library('form_validation');

        $this->form_validation->set_rules('name', 'Name', 'required|min_length[5]');

        if ($this->form_validation->run() == FALSE)
        {
          // Redirect with Errors or send Errors via JSON Response...
        }

        $room = $this->hipsupport->init(array(
          'room_name' => 'Live Chat with ' . $this->input->get('name'),
          'notification' => array(
            'message' => $this->input->get('name') . ' would like to chat.'
          )
        ));

        if ($room) 
        {
          if ($this->input->is_ajax_request())
          {
            echo json_encode(array('url' => $room->hipsupport_url));
            exit();
          }
          redirect($room->hipsupport_url);
        }
      }

    }

### Rate Limiting

HipChat's API currently limits API requests to 100 requests per 5 minutes. Each `$this->hipsupport->init()` call eats 3 requests (check if room name exists, create room and notify room). Read more on [HipChat's rate limiting](https://www.hipchat.com/docs/api/rate_limiting).

### Too Many Rooms!

It's probably a good idea to adopt some form of consistant naming of dynamically created rooms. In the next version of HipSupport I plan on added some more Artisan commands to help mass delete or mass archive rooms that have been inactive for a specified amount of time and whose room name contains a given string.