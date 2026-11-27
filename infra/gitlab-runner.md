# GitLab Runner

## 개념

### concurrent와 limit

설정 파일: `/etc/gitlab-runner/config.toml`


| 설정           | 위치                | 의미                                               |
| ------------ | ----------------- | ------------------------------------------------ |
| `concurrent` | 전역 (파일 최상단)       | 이 config.toml의 **모든 러너를 합친** 동시 실행 잡 상한          |
| `limit`      | `[[runners]]` 섹션별 | 그 러너 하나의 상한. `0`이면 러너별 제한 없음 (전역 concurrent만 적용) |


- 예: concurrent = 5, shell의 limit = 0, docker의 limit = 3
  - docker는 최대 3개, shell은 최대 5개, **둘을 합쳐서는 항상 5개 이하**
- 러너는 config.toml 변경을 감지해 자동으로 다시 읽는다. 재시작 없이 concurrent 변경이 반영된다.

### executor

- executor는 러너가 잡을 "어떤 환경에서" 실행할지 정하는 방식. `.gitlab-ci.yml`의 `script:`를 어디서 돌릴지를 결정한다.

```
GitLab 서버 ──(잡 요청)──→ GitLab Runner ──→ executor ──→ 실제 실행 환경
                           (잡을 가져옴)      (실행 방식)
                                              ├─ shell  → 호스트에서 바로 실행
                                              └─ docker → 새 컨테이너를 띄워 그 안에서 실행
```

- `config.toml`의 `[[runners]]` 섹션마다 `executor = "shell"`처럼 지정한다. 러너 하나에 executor 하나.
- 잡이 특정 러너로 가게 하려면 `.gitlab-ci.yml`의 `tags:`를 러너에 등록된 태그와 맞춘다.
- `image:`는 docker나 kubernetes executor에서만 의미가 있다. shell executor에서는 무시된다.

| executor | 실행 위치 | 장점 | 단점 |
| --- | --- | --- | --- |
| **shell** | 러너 호스트에서 `gitlab-runner` 사용자로 직접 실행 | 설정이 간단함. 호스트 도구를 바로 사용 | 잡끼리 격리 안 됨. 호스트 환경이 오염됨 |
| **docker** | 잡마다 `image:`로 지정한 컨테이너를 띄워 실행하고, 끝나면 제거 | 잡마다 깨끗하고 격리된 환경. 버전별 이미지 선택 | 이미지 받는 시간, 디스크 사용 |
| kubernetes | 잡마다 k8s Pod 생성 | 클러스터 단위로 확장 | 클러스터 운영이 필요 |
| ssh, virtualbox 등 | 원격 서버나 VM | 특수한 환경용 | 잘 안 쓰임 |

### executor별 격리 수준

|                                            | shell executor                   | docker executor                                      |
| ------------------------------------------ | -------------------------------- | ---------------------------------------------------- |
| 실행 위치                                      | 호스트에서 `gitlab-runner` 사용자로 직접 실행 | 잡마다 별도 컨테이너                                          |
| 홈 디렉터리 (`~/.docker`, `~/.m2`, `~/.gradle`) | **모든 잡이 공유**                     | 컨테이너별로 분리                                            |
| 포트, `/tmp`                                 | **공유** (충돌 가능)                   | 분리                                                   |
| CPU / 메모리                                  | 호스트 공유                           | 기본은 호스트 공유. `[runners.docker]`의 `memory`, `cpus`로 제한 |
| 작업 디렉터리 (`builds_dir`)                     | concurrent ID로 자동 분리             | 분리                                                   |


→ shell executor에서 concurrent를 올리면 **같은 호스트 자원을 동시에 쓰게 되므로** 충돌을 점검해야 한다.

### docker login 충돌

- `docker login`은 인증 정보를 `~/.docker/config.json` 한 파일에 저장한다.
- shell executor에서는 모든 잡이 같은 사용자라서 이 파일을 공유한다.
- 동시 실행 시 생기는 문제
  - 다른 잡이 다른 계정으로 login하면 인증 정보가 덮어써진다.
  - 다른 잡이 `docker logout`하면 push 중이던 잡이 실패한다.
  - 동시에 쓰면 파일이 손상될 수 있다.
- 해결: 잡마다 별도 설정 디렉터리를 쓴다.
  ```yaml
  variables:
    DOCKER_CONFIG: $CI_PROJECT_DIR/.docker
  ```

### 잡 타임아웃 (멈춘 잡의 근본 대책)

concurrent를 올리는 것은 **대기를 줄이는 완화책**이다. 멈춘 잡이 슬롯 수만큼 생기면 같은 문제가 재발한다. 근본 대책은 타임아웃이다.


| 계층     | 설정 위치                                                             | 역할                    |
| ------ | ----------------------------------------------------------------- | --------------------- |
| 잡      | `.gitlab-ci.yml`의 `timeout: 15m`                                  | 잡별 적정값                |
| 프로젝트   | Settings &gt; CI/CD &gt; General pipelines &gt; Timeout (기본값 60분) | 프로젝트 기본값              |
| **러너** | GitLab UI 러너 설정 &gt; Maximum job timeout                          | **프로젝트 설정보다 우선하는 상한** |


- 여러 프로젝트가 러너 하나를 공유하면 **러너의 maximum timeout이 핵심**이다. 각 프로젝트의 설정에 기대지 않고 관리자가 상한을 강제할 수 있다.
- pytest라면 `pytest-timeout`으로 테스트 하나 단위의 타임아웃도 걸 수 있다. 멈춘 지점의 스택 트레이스가 남는다.

### concurrent 값 정하기

코어 수나 메모리 총량이 아니라 **잡 하나가 실제로 쓰는 자원**이 기준이다.

1. 메모리: 잡 하나의 최대 사용량 × concurrent 값이 가용 메모리보다 넉넉히 작아야 한다.
2. 같은 서버의 다른 서비스(앱 컨테이너 등)가 쓸 몫을 먼저 남긴다.
3. 디스크 I/O와 용량 (Docker 이미지, 빌드 디렉터리 누적)
4. 공유 자원 충돌 (포트, Docker 데몬, 테스트 DB)

→ 조금씩 올리면서 메모리 `available`, 디스크, 앱 응답 시간, 잡 대기 시간을 모니터링한다.

---

## 장애 기록

### 한 줄 (채널·커밋 메모용)

- 11/27 개발서버 GitLab 러너 concurrent 1 → 5 상향(관리자 변경). 사유: uam_simul 잡 #30301이 멈춰 러너를 20분 이상 점유, 약 30개 프로젝트가 한 슬롯을 공유해 대기 발생.

### 249 GitLab 러너 동시 실행 수 변경 (운영 문서·공지용)

- 일시: 2026-11-27
- 대상: /etc/gitlab-runner/config.toml의 concurrent (러너 dev-was-shell, dev-was-docker 공통. 두 러너가 합쳐서 5개)
- 변경: 1 → 5
- 사유: uam_simul 잡 #30301의 pytest가 10:02(KST)부터 멈춰 20분 이상 러너를 점유했고, concurrent = 1이라 같은 러너를 쓰는 약 30개 프로젝트의 잡이 대기함(site_finder 잡은 25분 대기). 서버는 64코어, 메모리 가용 약 86GB.
- 백업: /etc/gitlab-runner/config.toml.bak-1127 (원래 값 concurrent = 1)
- 되돌리기: sudo sed -i 's/^concurrent = 5$/concurrent = 1/' /etc/gitlab-runner/config.toml (러너는 설정을 자동으로 다시 읽음, 서비스 재시작은 진행 중 잡을 끊을 수 있어 피함)
- 지켜볼 것: 서버 메모리·디스크, docker login 충돌로 인한 push 실패, 앱 컨테이너 응답 지연
- 후속 조치 (근본 대책): 러너 Maximum job timeout 설정(예: 15분), docker 사용 잡에 `DOCKER_CONFIG` 분리