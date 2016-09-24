ruby -rubygems -e 'require "jekyll-import";
    JekyllImport::Importers::WordPress.run({
      "dbname"   => "benwiener",
      "user"     => "root",
      "password" => "natasha",
      "host"     => "localhost",
      "table_prefix"   => "wp_"
    })'
