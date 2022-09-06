# Semgrep

高リスクなコードをSemgrepで抽出し、目視レビューで精査する。

## Strong Parameters

```bash
$ semgrep -e '$P.permit' --lang=ruby railsgoat/

Findings:

  railsgoat/app/controllers/messages_controller.rb
         42┆ params.require(:message).permit(:creator_id, :message, :read, :receiver_id)


  railsgoat/app/controllers/schedule_controller.rb
         62┆ params.require(:schedule).permit(:date_begin, :date_end, :event_desc, :event_name, :event_type)


  railsgoat/app/controllers/users_controller.rb
         57┆ params
         58┆   .require(:user)
         59┆
         60┆   .permit(:email, :admin, :first_name, :last_name)
```

## render json

```bash
$ semgrep -e '$P.to_json' --lang=ruby railsgoat/ 2> /dev/null

Findings:

  railsgoat/app/controllers/api/v1/mobile_controller.rb
         11┆ respond_with model.find(params[:id]).to_json
          ⋮┆----------------------------------------
         18┆ respond_with model.all.to_json
          ⋮┆----------------------------------------
         20┆ respond_with nil.to_json


  railsgoat/app/controllers/schedule_controller.rb
         37┆ format.json { render json: jfs.to_json }

```

## to_json

```bash
$ semgrep -e 'render(json:...)' --lang=ruby railsgoat/ 2>/dev/null

Findings:

  railsgoat/app/controllers/admin_controller.rb
         44┆ format.json { render json: { msg: message ? "success" : "failure"} }
          ⋮┆----------------------------------------
         57┆ format.json { render json: { msg: message ? "success" : "failure"} }


  railsgoat/app/controllers/messages_controller.rb
         29┆ format.json { render json: {msg: "success"} }
          ⋮┆----------------------------------------
         34┆ format.json { render json: {msg: "failure"} }


  railsgoat/app/controllers/pay_controller.rb
         18┆ format.json { render json: {msg: msg } }
          ⋮┆----------------------------------------
         24┆ format.json { render json: {user: current_user.pay.as_json} }
          ⋮┆----------------------------------------
         41┆ format.json { render json: {account_num: decrypted || "No Data" } }


  railsgoat/app/controllers/schedule_controller.rb
         18┆ format.json { render json: {msg: message ? "success" : "failure" } }
          ⋮┆----------------------------------------
         37┆ format.json { render json: jfs.to_json }


  railsgoat/app/controllers/users_controller.rb
         40┆ format.json { render json: {msg: message ? "success" : "false "} }
```