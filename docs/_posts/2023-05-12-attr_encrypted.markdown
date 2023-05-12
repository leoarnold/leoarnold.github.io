---
layout: post
title:  "From attr_encrypted to ActiveRecord Encryption in plain Ruby"
date:   2023-04-23 23:59:41 +0200
tags:
  - database
  - rails
---

```ruby
# frozen_string_literal: true

class ConvertAttrEncryptedToActiveRecordEncryption < ActiveRecord::Migration[7.0]
  KEYS = Rails.application.credentials.attr_encrypted

  ATTR_ENCRYPTED_ATTRIBUTES = {
    "CreditCard" => {
      public_account_number: { key: KEYS[:credit_cards] }
    },
    "User" => {
      external_oauth_token: { key: KEYS[:user], marshal: true }
    }
  }.freeze

  def up
    ATTR_ENCRYPTED_ATTRIBUTES.each do |class_name, attrs_encrypted|
      klass = Object.const_get(class_name)
      scope = klass.none

      attrs_encrypted.each_key do |attribute_name|
        scope = scope.or(klass.where.not(legacy_column_name(attribute_name) => nil))
      end

      scope.select(Arel.star).find_each do |row|
        old_attributes = row.attributes
        new_attributes = {}

        attrs_encrypted.each do |attribute_name, options|
          next if old_attributes[legacy_column_name(attribute_name)].blank?

          value = decrypt_attr_encrypted(old_attributes, legacy_column_name(attribute_name), options)

          new_attributes[attribute_name] = value
        end

        if new_attributes.present?
          record = klass.find(row.id)
          record.assign_attributes(new_attributes)
          record.save!(validate: false)
        end
      end
    end
  end

  private

  # Simplified from https://github.com/attr-encrypted/attr_encrypted/blob/3a9dfc78c3d24799f54a1926033d19e7448afe59/lib/attr_encrypted.rb
  def decrypt_attr_encrypted(old_attributes, attribute_name, options)
    decrypted_value = decrypt(
      iv: old_attributes.fetch("#{attribute_name}_iv").unpack1("m"),
      key: options.fetch(:key),
      value: old_attributes.fetch(attribute_name.to_s).unpack1("m")
    )

    options[:marshal] ? Marshal.load(decrypted_value) : decrypted_value # rubocop:disable Security/MarshalLoad
  end

  # Simplified from https://github.com/attr-encrypted/encryptor/blob/b9bb76039b7f3d61f62ec87f31329881ecede26c/lib/encryptor.rb#L54-L101
  def decrypt(iv:, key:, value:) # rubocop:disable Naming/MethodParameterName
    cipher = OpenSSL::Cipher.new("aes-256-gcm")
    cipher.decrypt

    raise ArgumentError, "key must be #{cipher.key_len} bytes or longer" if key.bytesize < cipher.key_len
    raise ArgumentError, "must specify an iv" if iv.to_s.empty?
    raise ArgumentError, "iv must be #{cipher.iv_len} bytes or longer" if iv.bytesize < cipher.iv_len

    cipher.key = key
    cipher.iv = iv
    cipher.auth_tag = value[-16..]
    cipher.auth_data = ""

    string = cipher.update(value[0..-17]) + cipher.final

    string.force_encoding("UTF-8")
  end

  def legacy_column_name(attribute_name)
    "encrypted_#{attribute_name}"
  end
end
```


[ar-encryption]: https://guides.rubyonrails.org/active_record_encryption.html
[attr_encrypted]: https://github.com/attr-encrypted/attr_encrypted
[encoding]: https://github.com/attr-encrypted/attr_encrypted/blob/dee8d41439ee6026889d2fb3308905c0e3c2cbc0/lib/attr_encrypted.rb#L246-L247
[encryptor]: https://github.com/attr-encrypted/encryptor
