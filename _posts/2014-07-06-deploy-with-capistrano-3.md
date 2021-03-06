---
layout: post
title: 创建与部署rails 4项目的全面流程
---

这个文档相关的例子项目在[这里](https://github.com/wolfgangjiang/cap3example)。

即便是天天写ruby on rails项目的程序员，在工作中接触“创建项目”和“冷部署”任务的机会也不很多。要想熟练地掌握这两门技能，最好自己在开发机上安装一个虚拟机，里面安装一个干净的linux操作系统，为这个干净的状态创建一个虚拟机快照，然后就可以反复练习了。

以下假定rails项目是托管在git上，部署到ubuntu服务器上，使用mongodb作为数据库，使用rspec和capybara作为自动测试框架，使用unicorn和nginx提供服务。

第一，创建项目
--------------

确认开发机上安装了足够新版本的rvm，要能支持.ruby-version与.ruby-gemset指示。

创建项目目录，在这里我们创建一个名叫cap3example的rails项目。先创建目录cap3example。

执行

    rvm --create --ruby-version 2.0.0@cap3example

这样，在cap3example目录下就出现了.ruby-version与.ruby-gemset文件。

为什么要在创建rails项目之前就设置好.ruby-version和.ruby-gemset？这是为了让项目使用独立的ruby版本与gemset，避免受到开发机上已有的ruby版本与rails版本的影响。在这个新建的gemset中，新安装最新稳定版本的rails，执行

    gem install rails

然后，使用这个新的gemset和新安装的rails，执行

    rails new -O -T cap3example

这里，`-O`是意思是省略active record，这是为了在项目中仅使用mongodb，不使用sql数据库。如果项目是需要使用sql数据库的，则不应该加上这个参数。

而`-T`参数的意思是省略test/unit，在后面如果需要的话，可以安装rspec和capybara用于自动测试。

然后执行

    mv .ruby* cap3example/

这样，项目的所有相关文件都处在cap3example/cap3example/目录下了。如果觉得cap3example/cap3example/这样的目录结构难看，可以在这个时候将子目录移动到上一级目录下。

第二，初始化项目
---------------

添加mongodb数据库支持，使用mongoid这个gem。在Gemfile中添加这样一行：

    gem 'mongoid', github: 'mongoid/mongoid'

并且执行

    bundle

使用github是因为mongoid的较旧的版本对rails 4的支持不好。当你看到此文时，可能mongoid已经更好地在稳定版中支持了rails 4。那时，这个例子中的版本应该指定为你创建项目时的当前版本。参见[mongoid官网](http://mongoid.org)。

创建mongoid.yml，格式如例子项目中的文件所示。然后在项目的.gitignore中添加一行

    /config/mongoid.yml

然后在config目录下执行

    cp mongoid.yml mongoid.yml.example

这是为了不把可能包含密码的数据库文件提交到公开的代码仓库中。而协作者在克隆这个项目时，可以从mongoid.yml.example中获取开发所需的配置文件的例子。

然后在Gemfile中添加

    gem 'slim'

以使用slim作为模版引擎。

在Gemfile中添加

    group :development, :test do
      gem 'rspec-rails', '~> 2.0'
      gem 'capybara'
    end

执行

    bundle

以安装rspec和capybara作为自动测试框架。

然后初始化rspec，执行

    rails generate rspec:install

如果没有使用active record，在rspec/rails_helper.rb中要注释
掉关于active record和fixtures的数行配置。

在rails_helper中加入一行

    require 'capybara/rails'

和一行

    config.include Capybara::DSL

最后，为了服务器上进行assets precompile，需要将Gemfile中的

    gem 'therubyracer'

一行去掉注释，并且再次执行

    bundle

这样就完成了对自动测试的配置。

第三，开发
----------

开发这个项目，写出业务所需的各种页面和功能，直到适合进行初
次部署的状态。

第四，在服务器上安装必要的软件
------------------------------

如果你的服务器是从一个过去创建的服务器快照上复制过来的，那
么可能其软件已经不是最新的了。第一步应该执行的是

    sudo apt-get update
    sudo apt-get dist-upgrade
    sudo reboot

这里，重启是因为很可能linux内核等核心软件也经过了更新，需要
重启以让新版本发挥作用。

然后要安装git、rvm、mongodb、nginx。

安装git的方式是：

    sudo apt-get install git

安装rvm的过程请参见其[官网](https://rvm.io)，执行

    \curl -sSL https://get.rvm.io | bash -s stable

然后执行

    rvm install 2.0.0

这样rvm和ruby 2.0就安装完成了。

安装mongodb请参见其[官网](http://docs.mongodb.org)，依次执行

    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
    echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list
    sudo apt-get update
    sudo apt-get install mongodb-org

这样mongodb也安装完成了。

安装nginx请参见其[官网](http://wiki.nginx.org)，依次执行

    sudo apt-get install python-software-properties
    sudo add-apt-repository ppa:nginx/stable
    sudo apt-get update
    sudo apt-get install nginx

这样nginx的安装也完成了。

需要将开发机的ssh public key拷贝到服务器上去，加入到~/.ssh/authorized_keys之中，使得capistrano能够不用密码就进行ssh访问，对服务器进行远程操作。

需要在服务器的根目录上创建/srv目录，并且执行（假定部署用的用户名为wolfgang）

    sudo chown wolfgang:wolfgang /srv

把这个目录设置为非root权限可写的。这样capistrano在整个部署过程中都可以以一般权限的部署用户操作，而不必获取服务器的root权限了，这样提高了安全性。

第五，在开发机上安装和配置capistrano 3与unicorn
-----------------------------------------------

capistrano 3与capistrano 2相比，对bundle、rvm等的支持都包装
得更好了。在远端执行命令时，使用的是execute方法，比2.x的
run方法加强了运行目录管理、变量嵌入与shell转义的功能。请参
见[官网](http://capistranorb.com)。

capistrano 3提供了capistrano-rails 1.0的包装。应该安装这个
gem，在Gemfile中添加：

    group :development do
      gem 'capistrano-ext'
      gem 'capistrano-rails', '~> 1.0.0'
      gem 'capistrano-bundler'
      gem 'capistrano-rvm'
    end

其中capistrano-bundler、capistrano-rvm等都是对我们很有用的
工具。执行

    bundle

进行安装。gem安装好了以后，在项目目录下执行

    cap install

对capistrano进行初始化。这个命令会生成Capfile、config/deploy.rb、config/deploy/production.rb、config/deploy/staging.rb等文件。这里，绝大部分要在服务器上执行的操作都在deploy.rb中定义。Capfile里需要添加一些require语句。而production.rb和staging.rb中是服务器ip地址等少量信息，每个此类文件负责指定一个服务器，使得capistrano可以有选择地部署在其中一个服务器上。

然后需要在Capfile中添加几个附属gems的require语句：

    require 'capistrano/bundler'
    require 'capistrano/rails/assets'
    require 'capistrano/rvm'

添加在require 'capistrano/deploy'这一行之后。

接下去修改config/deploy/production.rb，改为这样的内容：

    set :stage, :production

    set :rails_env, "production"
    set :application, "cap3example"
    set :branch, "master"
    set :user, "wolfgang"
    set :deploy_to, "/srv/#{fetch(:application)}"

    server "221.45.xxx.xxx", user: "wolfgang", roles: %w{web app db}

这里，application的赋值就将是在服务器上的/srv/目录下的子目录名字。而server这一行指定了服务器的ip地址与部署用户的用户名。

config/deploy/目录下，可以增加更多的服务器，比方说一个项目可能部署到20个不同的服务器上，则config/deploy/目录下就可以创建20个文件，每个文件指定一个服务器。每个文件的文件名就是capistrano所辨识的一个“部署点”。

下一步要做的是充实最核心的deploy.rb文件。以下的set内容需要定制：

    set :application, 'cap3example'
    set :repo_url, 'git@git.my-repo.com:cap3example.git'
    set :use_sudo, false
    set :deploy_timestamped, true
    set :release_name, Time.now.localtime.strftime("%Y%m%d%H%M%S")
    set :keep_releases, 5
    set :rvm_ruby_version, "2.0.0"

    set :linked_files, %w{.ruby-version .ruby-gemset config/mongoid.yml config/unicorn.rb config/secrets.yml}

    set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets public/system}

其余的set内容可以保留为注释的状态。

（注意要为production模式准备一个secret，储存在服务器上，加入.gitignore中，不提交到git仓库上。用rake secret命令可以获取一个新的secret。）

因为capistrano没有为app服务器提供预制的包装，而我们是使用unicorn作为app服务器，所以关于unicorn的配置需要额外添加到deploy.rb中。

在Gemfile中添加

    gem 'unicorn'

并且执行

    bundle

安装unicorn，作为app服务器。

将例子代码中的unicorn.rb拷贝到项目的config/目录下。其中需要
注意的几行是：

    worker_processes 2
    listen "#{APP_ROOT}/tmp/sockets/unicorn.sock", :backlog => 64
    pid "#{APP_ROOT}/tmp/pids/unicorn.pid"

而before_fork和after_fork可以都留空。

在capistrano的config/deploy.rb中添加如下的任务：

    task :start do
      on roles(:app) do
        within release_path do
          set :rvm_path, "~/.rvm"
          execute :bundle, "exec", "unicorn_rails", "-c", File.join(release_path, "config/unicorn.rb"), "-E production", "-D"
        end
      end
    end

    task :stop do
      on roles(:app) do
        pid_file = File.join(release_path, "tmp/pids/unicorn.pid")
        execute "if [[ -e #{pid_file} ]]; then kill $(cat #{pid_file}); fi"
      end
    end

    task :restart do
      invoke "deploy:stop"
      invoke "deploy:start"
    end

即可。这样，在服务器上启动unicorn服务的命令就是

    bundle exec unicorn_rails -c #{release_path}/config/unicorn.rb -E production -D

三个参数中，`-c`表示配置文件的路径，`-E`表示rails运行的环境模式，`-D`表示是以daemon形式启动。

第六，将项目推送到git仓库上
---------------------------

将git仓库地址填写到deploy.rb中。并且将服务器的ssh publickey添加到git仓库中，使得服务器可以访问到git仓库。也可以使用只读的https形式访问git仓库。

第七，进行冷部署
----------------

在项目目录下执行

    cap production deploy

第一次执行这个命令，会遇到“找不到.ruby-version文件”或类似的错误。这是因为我们在deploy.rb中设置了`linked_files`和`linked_dirs`参数。这时，我们应该用scp或类似的命令将这些文件拷贝到服务器上的/srv/cap3example/shared目录下，并创建其余可能需要的文件和目录，然后再次尝试。

第一次部署需要进行全新的bundle，所以需要比较长的时间。

通常到这里就可以完成初次部署了。如果出错，请参考前面的内容，看是否有什么环节没有正确安装。

第八，配置nginx
---------------

我们的app使用nginx作为web服务器，配置虚拟主机和端口。在第一次冷部署之后，需要为nginx添加必要的配置文件，使它能够找到我们项目的unicorn接口。

nginx配置文件的例子在例子项目的sys_confs下，名为nginx.conf。请把它拷贝到服务器的/etc/nginx/sites-available/目录下，改成适合你项目的名字，并且在/etc/nginx/sites-enabled/目录下建立对应的软链。

这个nginx配置文件中，需要为项目定制修改的主要有upstream的名字和地址、root的地址、端口号和server_name。其中upstream必须和unicorn.rb里所定义的listen sock一致。

根据需求和项目名字改成合适的值之后，保存nginx的网站配置文件，重启nginx。

第九，为unicorn配置init.d
-------------------------

有了init.d脚本，才可以使得服务器在重启时会自动启动unicorn，
而且这个脚本也可以为在服务器上手动重启unicorn提供方便，也可
以方便其它自动脚本来管理unicorn进程。

范例的init.d脚本在例子项目的`sys_conf/init-d`中，如果要用于其
它的项目，只需要改变脚本中的`APP_ROOT`变量的值即可。

部署之后，将这个脚本从`sys_conf`目录中复制到linux系统的
/etc/init.d/目录下，并且改名为`unicorn_cap3example`或者与你项
目相关的名字，这样就可以用命令

    sudo service unicorn_cap3example restart

来重启unicorn了。

第十，cron-apt、logrotate、monit
--------------------------------

这三个工具也是维护系统正常运行的重要元素。cron-apt能够每日
定时地更新系统中的安装包，打好安全补丁；logrotate会对日志文
件做打包处理，避免日志文件撑满硬盘；monit会在nginx或
unicorn进程消失后自动重启，保持服务器能够持续运转。

请阅读这些工具的官网，学习并安装它们。
