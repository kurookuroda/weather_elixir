weather_ex/
├── Dockerfile                              # 第0章: mix phx.gen.release --docker で生成
├── mix.exs                                 # 第2章でHTTPクライアント用の依存追加、第6章で編集
├── config/
│   ├── config.exs                          # 第0,7章で編集
│   └── runtime.exs                         # 第0章で生成、第6,9章で本番設定を追記
├── lib/
│   ├── weather_ex/                         # ビジネスロジック層(Web非依存)
│   │   ├── application.ex                  # 第3章: StationCacheをSupervisorツリーに登録
│   │   └── weather/
│   │       ├── observation.ex              # 第1章: 純粋関数によるデータ変換
│   │       ├── jma_client.ex               # 第2章: 気象庁API連携
│   │       └── station_cache.ex            # 第3章: GenServerによる観測地点キャッシュ
│   │   └── weather.ex                      # 第4章: Context層(公開API)
│   └── weather_ex_web/                     # Web層
│       ├── router.ex                       # 第5章: /weather ルート追加
│       ├── endpoint.ex                     # 第10章: 編集(静的アセット配信設定など)
│       └── live/
│           ├── weather_live.ex             # 第5章生成、第7,8章で拡張(JS Hook連携、Geolocation)
│           └── weather_live.html.heex      # 第5章生成、第7,8,10章でテンプレート拡張
├── assets/
│   ├── css/
│   │   └── app.css                         # 第7章: Leaflet用スタイル追加
│   └── js/
│       ├── app.js                          # 第7章: Hooksの登録
│       └── hooks/
│           └── weather_map.js              # 第7章生成、第8,10章で拡張(Geolocation, 鯨アニメーション)
├── priv/
│   └── static/
│       └── images/
│           ├── whale_spritesheet.png       # 第10章(任意): 鯨アニメーション用スプライト画像
│           └── whale_spritesheet.json      # 第10章(任意): スプライトのメタデータ
├── rel/
│   └── overlays/
│       └── bin/
│           ├── server                      # 第0章: リリース起動スクリプト
│           └── server.bat
└── test/
    └── weather_ex/
    │   └── weather/
    │       ├── observation_test.exs        # 第1章
    │       ├── jma_client_test.exs         # 第2章
    │       └── station_cache_test.exs      # 第3章
    └── weather_ex_web/
        └── live/
            └── weather_live_test.exs       # 第5章
