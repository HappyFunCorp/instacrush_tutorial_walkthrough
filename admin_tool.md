# Admin Tool

Ok, so we've mocked out the UI, lets go through and add a quick admin tool so that we can see what's going on.

First, let's run the generator and commit the changes.

```
$ rails g happy_seed:admin
$ rake db:migrate
$ git add .
$ git commit -a -m "Applied happy_seed:admin"
```

Lets go to the console now, and create a basic admin user.

```
$ rails c
Loading development environment (Rails 4.2.4)
[1] instacrush_tutorial »  AdminUser.create!(email: 'admin@example.com', password: 'password', password_confirmation: 'password')
```

Restart your server, and go to http://localhost:3000/admin

This is what you should see:

![](Login___Instacrush_Tutorial.jpg)

Enter in `admin@example.com` and `password`, or whatever you chose to generate above.

Here's the default dashboard:

![](Dashboard___Instacrush_Tutorial.jpg)

## Changing the charts

First thing we'll do is to change what is displayed on the charts.  This is defined in the file app/admin/dashboard.rb

Lets add a few more panels in there, so that we can graph out some highlevel things we're interested in seeing.

```
ActiveAdmin.register_page "Dashboard" do

  menu priority: 1, label: proc{ I18n.t("active_admin.dashboard") }

  content title: proc{ I18n.t("active_admin.dashboard") } do
    columns do
      column do
        panel "Users" do
          render partial: "admin/chart", locals: { scope: 'users' }
        end
      end
    end

    columns do
      column do
        panel "Instagram Users" do
          render partial: "admin/chart", locals: { scope: 'instagram_users' }
        end
      end
      column do
        panel "Crushes" do
          render partial: "admin/chart", locals: { scope: 'crushes' }
        end
      end
    end

    columns do
      column do
        panel "Likes" do
          render partial: "admin/chart", locals: { scope: 'likes' }
        end
      end
      column do
        panel "Comments" do
          render partial: "admin/chart", locals: { scope: 'comments' }
        end
      end
    end
  end
end
```

Now we need to change the admin/stats controller, in `app/controllers/admin/stats_controller.rb` to return the right data for the scopes that we pass in.

`