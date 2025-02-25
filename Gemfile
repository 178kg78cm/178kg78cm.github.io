# frozen_string_literal: true

source "https://rubygems.org"

# GitHub Pages에서 Jekyll을 사용할 경우, 지원되는 버전으로 설정
gem "github-pages", "~> 232", group: :jekyll_plugins

# 기본적인 Ruby 라이브러리 추가 (Ruby 3.4.0 이후 필수)
gem "csv"
gem "base64"
gem "bigdecimal"

# Sass 관련 패키지 (SCSS 변환 오류 해결)
gem "sassc"

# Windows 환경에서 파일 변경 감지를 최적화
gem "wdm", ">= 0.1.0" if Gem.win_platform?
