stage('Stage III: SCA (OWASP, JDK17)') {
  steps {
    echo "Running Software Composition Analysis using OWASP Dependency-Check ..."
    withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
      sh '''
        set -e
        export JAVA_HOME="$JAVA17"; export PATH="$JAVA_HOME/bin:$PATH"

        # --- CONFIG (user-writable paths) ---
        PERSIST_DC_DIR="${HOME:-$WORKSPACE}/.odc-cache"   # persistent cache per agent user
        USE_DC_DIR="$PERSIST_DC_DIR"
        SEED_URL=""   # OPTIONAL: set to your odc-cache.tgz URL when you have one

        # 0) Ensure key
        if [ -z "$NVD_API_KEY" ]; then
          echo "ERROR: NVD_API_KEY not set (check Jenkins credentials ID)."
          exit 9
        fi
        NVD_API_KEY_CLEAN="$(printf "%s" "$NVD_API_KEY" | tr -d "\\r\\n")"

        mkdir -p "$USE_DC_DIR"
        if [ ! -w "$USE_DC_DIR" ]; then
          echo "ERROR: Cache dir $USE_DC_DIR not writable."
          exit 12
        fi

        # 1) Probe API (should be 200)
        CODE=$(curl -s -o /dev/null -w "%{http_code}" \
          -H "apiKey: $NVD_API_KEY_CLEAN" \
          "https://services.nvd.nist.gov/rest/json/cves/2.0?resultsPerPage=1")
        echo "NVD connectivity HTTP code: $CODE"

        # 2) Try throttled update-only into the persistent cache (best effort)
        UPDATE_OK=0
        if [ "$CODE" = "200" ]; then
          echo "Attempting throttled update-only into $USE_DC_DIR ..."
          set +e
          mvn -B \
            -DskipTests \
            -DdataDirectory="$USE_DC_DIR" \
            -Dnvd.api.key="$NVD_API_KEY_CLEAN" \
            -Dnvd.api.delay=15000 \
            -Dnvd.api.maxRetryCount=10 \
            -Dnvd.api.retryDelay=12000 \
            -Dnvd.api.cvesPerPage=120 \
            -Dnvd.api.startYear=2018 \
            org.owasp:dependency-check-maven:update-only
          EC=$?
          set -e
          if [ $EC -eq 0 ]; then
            UPDATE_OK=1
            echo "Update-only completed."
          else
            echo "WARNING: update-only failed with exit code $EC (likely NVD throttling)."
          fi
        else
          echo "WARNING: NVD probe failed ($CODE); will try offline options."
        fi

        # 3) If update failed and no cache exists yet, optionally fetch a pre-seeded cache
        if [ $UPDATE_OK -ne 1 ] && ! ls "$USE_DC_DIR"/*.mv.db >/dev/null 2>&1; then
          if [ -n "$SEED_URL" ]; then
            echo "No local cache; fetching pre-seeded cache from: $SEED_URL"
            set +e
            curl -fsSL "$SEED_URL" -o /tmp/odc-cache.tgz
            T_EC=$?
            set -e
            if [ $T_EC -ne 0 ]; then
              echo "ERROR: Could not download pre-seeded cache (HTTP error)."
              echo "Proceeding by SKIPPING SCA for this run."
              exit 0   # <-- lenient: skip SCA, keep pipeline green
            fi
            tar xzf /tmp/odc-cache.tgz -C "$USE_DC_DIR" --strip-components=1 || true
            echo "Pre-seeded cache unpacked."
          else
            echo "WARNING: No cache present and SEED_URL not set; SKIPPING SCA for this run."
            exit 0     # <-- lenient: skip SCA, keep pipeline green
          fi
        fi

        # 4) Run analysis strictly offline (no more API calls)
        # Also lenient on errors: -DfailOnError=false to avoid failing the build
        mvn -B \
          -DskipTests \
          -Dautoupdate=false \
          -DfailOnError=false \
          -DdataDirectory="$USE_DC_DIR" \
          -Dnvd.api.key="$NVD_API_KEY_CLEAN" \
          -Dformat=HTML \
          -DoutputDirectory=target/dependency-check \
          org.owasp:dependency-check-maven:check || {
            echo "Dependency-Check encountered errors; continuing build (failOnError=false)."
            exit 0
          }
      '''
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'target/dependency-check/*', fingerprint: true
    }
  }
}
