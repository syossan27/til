# Unknown validator: 'EmailValidator'と怒られた

sorceryで普通にUserと紐付けたら掲題通り怒られた

```
validates :email, presence: true, uniqueness: true, allow_blank: true, email: true, length: { maximum: 255 }
```

のemail: trueが示しているようにCustomValidatorとしてEmailValidatorを探しているが無いと言う感じ。

gemの[email_validator](https://github.com/balexand/email_validator)を入れれば解決。
