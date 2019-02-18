# Steps

- rails new mack-child-muse
- rails g scaffold Post title link description:text
- rails db:migrate
- routes: root 'posts#index'
- haml gem
- simple form gem
- devise gem
- paperclip
- rails generate simple_form:install
- rails g devise:install
- rails g devise:views
- rails g devise User
- rails db:migrate
- rails g migration AddUserIdToPosts user:references
- update post.rb

```
belongs_to :user
```

- user.rb

```
has_many :posts
```

- update posts controller to build from current user

```
  def new
    @post = current_user.posts.build
  end

  # POST /posts
  # POST /posts.json
  def create
    @post = current_user.posts.build(post_params)

    respond_to do |format|
      if @post.save
        format.html { redirect_to @post, notice: 'Post was successfully created.' }
        format.json { render :show, status: :created, location: @post }
      else
        format.html { render :new }
        format.json { render json: @post.errors, status: :unprocessable_entity }
      end
    end
  end
```

- and add before action authenticate and update params

```
before_action :authenticate_user!, except: [:index, :show]
def post_params
  params.require(:post).permit(:title, :link, :description, :user_id)
end
```

- create user
- and create a post
- rails g migration AddNameToUsers name
- rails db:migrate
- update new and edit registrations

```
<%= f.input :name, required: true, autofocus: true %>
```

- add the update to application controller

```
  before_action :configure_permitted_parameters, if: :devise_controller?

  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name])
    devise_parameter_sanitizer.permit(:account_update, keys: [:name])
  end
```

### Active storage

- rails active_storage:install
- rails db:migrate
- add to post.rb

```
has_one_attached :image
```

- update post params

```
def post_params
  params.require(:post).permit(:title, :link, :description, :user_id, :image)
end
```  

- update the form partial

```
  <div class="field">
    <%= form.label :image %>
    <%= form.file_field :image %>
  </div>
```

- update show page

```
= link_to (image_tag url_for(post.image)), post
```

### Adding comments

- rails g model Comment content:text post:references user:references
- rails db:migrate
- update post.rb

```
has_many :comments, dependent: :destroy
```

- update user.rb

```
has_many :comments, dependent: :destroy
```

- update routes

```
  resources :posts do 
    resources :comments
  end
```

- rails g controller Comments

```
class CommentsController < ApplicationController
	before_action :authenticate_user!
	def create
		@post = Post.find(params[:post_id])
		@comment = Comment.create(params[:comment].permit(:content))
		@comment.user_id = current_user.id
		@comment.post_id = @post.id

		if @comment.save
			redirect_to post_path(@post)
		else
			render 'new'
		end
	end
end
```

- create the comments/form partial

```
= simple_form_for([@post, @post.comments.build]) do |f|
	= f.input :content, label: "Reply to thread"
	= f.button :submit, class: "button"
```

- update the posts show to show the comments

```
#post_show
	%h1= @post.title
	%p.username
		Shared by
		= @post.user.name
		about
		= time_ago_in_words(@post.created_at)
	.clearfix
		.post_image_description
			= image_tag @post.image
			.description= simple_format(@post.description)
		.post_data
			= link_to "Visit Link", @post.link, class: "button"

			%p.data
				%i.fa.fa-comments-o
				= pluralize(@post.comments.count, "Comment")
			- if @post.user == current_user
				= link_to "Edit", edit_post_path(@post), class: "data"
				= link_to "Delete", post_path(@post), method: :delete, data: { confirm: "Are you sure?" }, class: "data"

#comments
	%h2.comment_count= pluralize(@post.comments.count, "Comment")
	- @comments.each do |comment|
		.comment
			%p.username= comment.user.name
			%p.content= comment.content

	= render "comments/form"
```

- add the comments to the show action in posts controller

```
@comments = Comment.where(post_id: @post)
```

## Adding the ability to like a post using acts as votable

- rails generate acts_as_votable:migration
- rails db:migrate
- add to the post.rb up top

```
acts_as_votable
```

- update routes

```
  resources :posts do 
  	member do
  		get "like", to: "posts#upvote"
  		get "dislike", to: "posts#downvote"
  	end
    resources :comments
  end
```

- add the upvote and downvote actions to the posts controller

```
	def upvote
		@post.upvote_by current_user
		redirect_back(fallback_location: root_path)
	end

	def downvote
		@post.downvote_from current_user
		redirect_back(fallback_location: root_path)
	end
```

- update the before action

```
before_action :set_post, only: [:show, :edit, :update, :destroy, :upvote, :downvote]
```

- add the links to the post/show

```
.post_data
	= link_to "Visit Link", @post.link, class: "button"
	= link_to like_post_path(@post), method: :get, class: "data" do
		%i.fa.fa-thumbs-o-up
		= pluralize(@post.get_upvotes.size, "Like")
	= link_to dislike_post_path(@post), method: :get, class: "data" do
		%i.fa.fa-thumbs-o-down
		= pluralize(@post.get_downvotes.size, "Dislike")
```				

## Add structure and styling

- update the layout/app to haml

```
!!!
%html
%head
	%title Muse
  = stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track' => true
  = javascript_include_tag 'application', 'data-turbolinks-track' => true
	%link{:rel => "stylesheet", :href => "http://cdnjs.cloudflare.com/ajax/libs/normalize/3.0.1/normalize.min.css"}
	%link{:rel => "stylesheet", :href => "http://maxcdn.bootstrapcdn.com/font-awesome/4.2.0/css/font-awesome.min.css"}
	= csrf_meta_tags
%body
	%header
		.wrapper.clearfix
			#logo= link_to "Muse", root_path
			%nav
				- if user_signed_in?
					= link_to current_user.name, edit_user_registration_path
					= link_to "Add New Inspiration", new_post_path, class: "button"
				- else
					= link_to "Sign in", new_user_session_path
					= link_to "Sign Up", new_user_registration_path, class: "button"
	%p.notice= notice
	%p.alert= alert
	.wrapper
		= yield
```

## Showing a random post on the show page

- update the show action in post controller

```
@random_post = Post.where.not(id: @post).order("RANDOM()").first
```

- update the show file

```
#post_show
	%h1= @post.title
	%p.username
		Shared by
		= @post.user.name
		about
		= time_ago_in_words(@post.created_at)
	.clearfix
		.post_image_description
			= image_tag @post.image
			.description= simple_format(@post.description)
		.post_data
			= link_to "Visit Link", @post.link, class: "button"
			= link_to like_post_path(@post), method: :get, class: "data" do
				%i.fa.fa-thumbs-o-up
				= pluralize(@post.get_upvotes.size, "Like")
			= link_to dislike_post_path(@post), method: :get, class: "data" do
				%i.fa.fa-thumbs-o-down
				= pluralize(@post.get_downvotes.size, "Dislike")
			%p.data
				%i.fa.fa-comments-o
				= pluralize(@post.comments.count, "Comment")
			- if @post.user == current_user
				= link_to "Edit", edit_post_path(@post), class: "data"
				= link_to "Delete", post_path(@post), method: :delete, data: { confirm: "Are you sure?" }, class: "data"
		#random_post
			%h3 Random Inspiration
			.post
				.post_image
					= link_to (image_tag url_for(@random_post.image)), post_path(@random_post)
				.post_content
					.title
						%h2= link_to @random_post.title, post_path(@random_post)
					.data.clearfix
						%p.username
							Shared by
							= @random_post.user.name
						%p.buttons
							%span
								%i.fa.fa-comments-o
								= @random_post.comments.count
							%span
								%i.fa.fa-thumbs-o-up
								= @random_post.get_likes.size

#comments
	%h2.comment_count= pluralize(@post.comments.count, "Comment")
	- @comments.each do |comment|
		.comment
			%p.username= comment.user.name
			%p.content= comment.content

	= render "comments/form"
```

## The End