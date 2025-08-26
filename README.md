# CarbonCreditNFTs.clar
;; ------------------------------------------------------------
;; Carbon Credit NFT Contract
;; ------------------------------------------------------------

(define-non-fungible-token carbon-credit uint)

;; Data map for NFT metadata
(define-map token-metadata
  uint
  {
    co2-amount: uint,
    project-name: (string-ascii 64),
    verification-authority: (string-ascii 32),
    issue-date: uint,
    retired: bool
  }
)

;; Contract variables
(define-data-var last-token-id uint u0)
(define-data-var total-credits-issued uint u0)
(define-data-var total-credits-retired uint u0)
(define-data-var contract-owner principal tx-sender)
(define-data-var contract-paused bool false)

;; Error codes
(define-constant err-owner-only (err u100))
(define-constant err-not-owner (err u101))
(define-constant err-invalid-amount (err u102))
(define-constant err-already-retired (err u103))
(define-constant err-contract-paused (err u104))

;; ------------------------------------------------------------
;; PRIVATE: internal mint (used by public + batch mint)
;; ------------------------------------------------------------
(define-private (internal-mint-credit
    (recipient principal)
    (co2-amount uint)
    (project-name (string-ascii 64))
    (verification-authority (string-ascii 32)))
  (let ((token-id (+ (var-get last-token-id) u1)))
    (begin
      (asserts! (not (var-get contract-paused)) err-contract-paused)
      (asserts! (> co2-amount u0) err-invalid-amount)

      (try! (nft-mint? carbon-credit token-id recipient))

      (map-set token-metadata token-id {
        co2-amount: co2-amount,
        project-name: project-name,
        verification-authority: verification-authority,
        issue-date: stacks-block-height,
        retired: false
      })

      (var-set last-token-id token-id)
      (var-set total-credits-issued (+ (var-get total-credits-issued) co2-amount))

      (ok token-id)
    )
  )
)

;; ------------------------------------------------------------
;; PUBLIC: single mint
;; ------------------------------------------------------------
(define-public (mint-carbon-credit
    (recipient principal)
    (co2-amount uint)
    (project-name (string-ascii 64))
    (verification-authority (string-ascii 32)))
  (begin
    (asserts! (is-eq tx-sender (var-get contract-owner)) err-owner-only)
    (internal-mint-credit recipient co2-amount project-name verification-authority)
  )
)

;; ;; ------------------------------------------------------------
;; ;; PUBLIC: batch mint
;; ;; ------------------------------------------------------------
;; (define-public (batch-mint-credits
;;     (mints (list 50 {recipient: principal, amount: uint}))
;;     (project-name (string-ascii 64))
;;     (verification-authority (string-ascii 32)))
;;   (begin
;;     (asserts! (is-eq tx-sender (var-get contract-owner)) err-owner-only)
;;     (let (
;;           (all-success
;;             (fold
;;               (lambda ((item {recipient: principal, amount: uint}) (context-ok bool))
;;                 (if (and context-ok
;;                          (is-ok (internal-mint-credit (get recipient item)
;;                                                       (get amount item)
;;                                                       project-name
;;                                                       verification-authority)))
;;                     true
;;                     false))
;;               true
;;               mints)))
;;       (ok all-success)
;;     )
;;   )
;; )

;; ------------------------------------------------------------
;; ;; Transfer NFT
;; ;; ------------------------------------------------------------
;; (define-public (transfer-credit (token-id uint) (recipient principal))
;;   (begin
;;     (asserts! (is-some (nft-get-owner? carbon-credit token-id)) err-not-owner)
;;     (nft-transfer? carbon-credit token-id tx-sender recipient)
;;     (ok true)
;;   )
;; )

;; ------------------------------------------------------------
;; Retire NFT
;; ------------------------------------------------------------
(define-public (retire-credit (token-id uint))
  (begin
    (asserts! (is-some (nft-get-owner? carbon-credit token-id)) err-not-owner)
    (let ((owner (unwrap! (nft-get-owner? carbon-credit token-id) err-not-owner)))
      (let ((metadata (unwrap! (map-get? token-metadata token-id) (err u105))))
        (asserts! (not (get retired metadata)) err-already-retired)

        (map-set token-metadata token-id (merge metadata { retired: true }))
        (var-set total-credits-retired (+ (var-get total-credits-retired) (get co2-amount metadata)))

        ;; (nft-burn? carbon-credit token-id)
        (print {retired-by: owner, token: token-id})
      )
    )
    (ok true)
  )
)

;; ------------------------------------------------------------
;; Owner controls
;; ------------------------------------------------------------
(define-public (pause-contract)
  (begin
    (asserts! (is-eq tx-sender (var-get contract-owner)) err-owner-only)
    (var-set contract-paused true)
    (ok true)
  )
)

(define-public (unpause-contract)
  (begin
    (asserts! (is-eq tx-sender (var-get contract-owner)) err-owner-only)
    (var-set contract-paused false)
    (ok true)
  )
)

;; ------------------------------------------------------------
;; Read-only getters
;; ------------------------------------------------------------
(define-read-only (get-token-owner (token-id uint))
  (nft-get-owner? carbon-credit token-id)
)

(define-read-only (get-token-metadata (token-id uint))
  (map-get? token-metadata token-id)
)

(define-read-only (get-total-issued)
  (ok (var-get total-credits-issued))
)

(define-read-only (get-total-retired)
  (ok (var-get total-credits-retired))
)
