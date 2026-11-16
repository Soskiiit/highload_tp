 flowchart TD
        %% ===== Клиенты и периметр =====
        subgraph S_EDGE["Входной периметр"]
            Clients["Клиенты (Mobile KMP / Web SPA)<br/>+ SQLite-кэш на клиенте"]
            CDN["Edge CDN (CloudFront)<br/>раздача TTS-аудио и статики"]
            Ingress["L4 IPVS + L7 Nginx<br/>(TLS 1.3, WAF)"]
            APIGW["API Gateway (BFF, Go)<br/>authN, rate limit, маршрутизация"]
            
            Clients --> Ingress
            Clients --> CDN
            CDN -.->|"origin miss (R)"| S3Content
            Ingress --> APIGW
            Ingress -->|"speech/*"| Ingest
            APIGW -->|"маршрутизация API"| S_ONLINE
        end

        %% ===== Онлайн-контур =====
        subgraph S_ONLINE["ОНЛАЙН-КОНТУР (Kubernetes EKS)"]

            subgraph S_AUTH["Auth Service"]
                AuthAPI["Auth API<br/>login / регистрация / токены"]
                AuthConsumer["Auth Consumer"]
                PostgresAuth[("PostgreSQL Auth<br/>аккаунты, password_hash")]
                RedisSessions[("Redis Sessions<br/>sess:token (TTL 30d)")]
                
  	        AuthAPI -->|"R/W"| PostgresAuth
                AuthAPI -->|"R/W"| RedisSessions
                AuthConsumer -->|"W (DEL sess)"| RedisSessions
            end

            subgraph S_USERS["Users Service"]
                UsersAPI["Users API (композиция)<br/>GET /user/progress (главная)"]
                UsersConsumer["Users Consumer"]
  		PostgresUsers[("PostgreSQL Users<br/>профиль, настройки, локаль")]
                UsersCache["Кэш профиля"]
                
  	        UsersAPI -->|"R/W"| PostgresUsers
                UsersAPI -->|"R/W"| UsersCache
                UsersConsumer -->|"W (инвалидация)"| UsersCache
                UsersConsumer -->|"W (удаление)"| PostgresUsers
            end

            subgraph S_PROGRESS["Progress & Practice Service"]
                LessonAPI["Lessons API (композиция)<br/>GET /lessons/:id, /mistakes<br/>POST /submit (атомарная запись)"]
                HLRRanker["HLR-ранжер (CPU)<br/>фичи, веса θ, min-heap Top-K"]
  		OutboxRelay["Outbox Relay"]
  		LessonCache["Fallback-кэш уроков"]
                ScyllaProgress[("ScyllaDB Progress<br/>progress, mistakes, θ, outbox<br/>PK: user_id")]
                
                LessonAPI --> HLRRanker
                LessonAPI -->|"R (прогресс)"| ScyllaProgress
                LessonAPI -->|"W (progress + outbox)"| ScyllaProgress
                HLRRanker -->|"R (mistakes, θ)"| ScyllaProgress
                OutboxRelay -->|"R (outbox)"| ScyllaProgress
                LessonAPI -->|"W (кэш сборки)"| LessonCache
  		LessonAPI -.->|"fallback R"| LessonCache
            end

            subgraph S_GAMIF["Gamification & Leagues Service"]
                GamifAPI["Leaderboard API (композиция)<br/>GET /leaderboard, /gamification"]
  		GamifConsumer["Gamification Consumer"]
                RedisGamif[("Redis Gamification<br/>XP/streak (HASH), лиги (ZSET)")]
                ScyllaGamif[("ScyllaDB Gamification<br/>xp_ledger (durable log)")]
	        
                GamifConsumer -->|"W (HINCRBY, ZADD)"| RedisGamif
                GamifConsumer -->|"W (ledger upsert)"| ScyllaGamif
                GamifAPI -->|"R (Top-30)"| RedisGamif
  		GamifAPI -.->|"fallback R (отказ Redis)"| ScyllaGamif
            end

            subgraph S_CONTENT["Course Content Service"]
                ContentAPI["Content API<br/>каталог, структура курсов"]
                PublishAPI["Publish API<br/>публикация + outbox"]
                MediaWorker["Media-воркер<br/>TTS (Amazon Polly) + ассеты"]
                PostgresContent[("PostgreSQL Content<br/>курсы, задания, tts_meta")]
                S3Content[("S3 Content<br/>TTS Opus + статика")]
                
                ContentAPI -->|"R"| PostgresContent
                PublishAPI -->|"W (контент + outbox)"| PostgresContent
                MediaWorker -->|"W (медиа)"| S3Content
                MediaWorker -->|"W (метаданные)"| PostgresContent
                MediaWorker -.->|"push / invalidate"| CDN
            end
    
            subgraph S_SEARCH["Search Service"]
                SearchAPI["Search API<br/>GET /courses (поиск)"]
                Indexer["Воркер-индексатор"]
                OpenSearch[("OpenSearch<br/>индекс каталога")]
                
                SearchAPI -->|"R"| OpenSearch
                Indexer -->|"W (индекс)"| OpenSearch
            end
    
            subgraph S_SPEECH["Speech Verification Service (GPU EC2)"]
                Ingest["Приём аудио<br/>+ декодер libopus"]
                Conformer["Conformer-CTC (TensorRT)<br/>+ forced alignment & GoP"]
                Sampler["Сэмплер 0,5%<br/>99,5% стирается в RAM"]
                ScyllaSpeech[("ScyllaDB Speech<br/>метаданные, TTL 90d")]
                S3Speech[("S3 Speech<br/>аудиозаписи, TTL 90d")]
                
                Ingest --> Conformer --> Sampler
            end
    
            subgraph S_EVENTS["Events Service"]
                EventsAPI["Events API<br/>POST /events (фронт-батчи)"]
            end
        end
    
        %% ===== Межсервисные связи (Синхронные вызовы композиции) =====
        UsersAPI -->|"R: очки / стрик"| GamifAPI
        UsersAPI -->|"R: прогресс"| LessonAPI
        LessonAPI -->|"R: упражнения"| ContentAPI
        LessonAPI -->|"R: жизни / лимиты"| GamifAPI
        GamifAPI -->|"R: профили Top-30"| UsersAPI
    
        %% ===== Асинхронная хореография между сервисами =====
        AuthAPI -.->|"auth_events (через Kafka)"| UsersConsumer
        UsersAPI -.->|"account_events (через Kafka)"| AuthConsumer
        OutboxRelay -.->|"lesson_submitted (через Kafka)"| GamifConsumer
        PublishAPI -.->|"content_events (через Kafka)"| MediaWorker
        PublishAPI -.->|"content_events (через Kafka)"| Indexer
    
        %% ===== Событийная шина (строго сверху вниз) =====
        subgraph S_KAFKA["Событийная шина (Apache Kafka KRaft)"]
            KafkaBus["Топики событий:<br/>• frontend_events (клиентские клики, сессии)<br/>• lesson_submitted (прогресс уроков)<br/>• speech_raw (0,5% аудио сэмплов)"]
        end
    
        EventsAPI -->|"фронт-события"| KafkaBus
        OutboxRelay -->|"бэк-события"| KafkaBus
        Sampler -->|"голос 0,5%"| KafkaBus
    
        %% ===== Оффлайн-контур =====
        subgraph S_OFFLINE["ОФФЛАЙН / NEAR-LINE КОНТУР"]
            subgraph S_DWH["DWH / Аналитика"]
                DWHConsumer["DWH-консьюмер<br/>батчи 20–50k"]
                Reconcile["Сверка front ↔ back<br/>+ ML-витрины"]
                ClickHouse[("ClickHouse DWH<br/>dwh_events")]
                
                DWHConsumer -->|"W (батчи)"| ClickHouse
                Reconcile -->|"R"| ClickHouse
            end

            HLRTrainer["HLR-тренер (Spark / PyTorch)<br/>ежесуточный пересчёт весов"]
            MistakesBuilder["Mistakes Builder<br/>near-line срез ошибок"]
            SpeechArchiver["Speech Archiver"]
            LeagueFinalizer["League Finalizer<br/>недельный пересчёт лиг"]

            KafkaBus -->|"события DWH"| DWHConsumer
            KafkaBus -->|"speech_raw"| SpeechArchiver
            
            HLRTrainer -->|"R"| ClickHouse
            HLRTrainer -->|"W (веса θ)"| ScyllaProgress
            MistakesBuilder -->|"R"| ClickHouse
            MistakesBuilder -->|"W (user_mistakes)"| ScyllaProgress
            SpeechArchiver -->|"W (аудио)"| S3Speech
            SpeechArchiver -->|"W (метаданные)"| ScyllaSpeech
            LeagueFinalizer -->|"R/W (когорты)"| RedisGamif
        end

        %% ===== Наблюдаемость (компактно) =====
        subgraph S_OBS["Контур наблюдаемости"]
            OtelAgent["OTel Collector + Vector"]
            ObsStorage[("VictoriaMetrics • Loki • Tempo")]
            Grafana["Grafana + Alertmanager"]

            OtelAgent --> ObsStorage --> Grafana
        end

        S_ONLINE -.->|"телеметрия"| OtelAgent
        S_OFFLINE -.->|"телеметрия"| OtelAgent
