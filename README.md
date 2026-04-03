BRIAN-TECH 
const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require("@whiskeysockets/baileys")
const P = require("pino")
const qrcode = require("qrcode-terminal") // NEW

let antiVideoSettings = {}

async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState("auth_info")

    const sock = makeWASocket({
        auth: state,
        logger: P({ level: "silent" }) // remove deprecated printQRInTerminal
    })

    sock.ev.on("creds.update", saveCreds)

    sock.ev.on("connection.update", (update) => {
        const { connection, lastDisconnect, qr } = update

        if (qr) {
            console.log("📱 Scan the QR code below with WhatsApp:")
            qrcode.generate(qr, { small: true }) // 🔥 show QR in terminal
        }

        if (connection === "close") {
            const reason = lastDisconnect?.error?.output?.statusCode

            if (reason !== DisconnectReason.loggedOut) {
                console.log("🔁 Reconnecting...")
                startBot()
            } else {
                console.log("❌ Logged out. Delete auth_info folder and restart bot.")
            }
        } else if (connection === "open") {
            console.log("✅ Bot connected successfully!")
        }
    })

    sock.ev.on("messages.upsert", async ({ messages }) => {
        const msg = messages[0]
        if (!msg.message) return

        const from = msg.key.remoteJid
        const sender = msg.key.participant || from
        const isGroup = from.endsWith("@g.us")

        const text =
            msg.message.conversation ||
            msg.message.extendedTextMessage?.text ||
            ""

        // ================= COMMAND HANDLER =================
        if (text.startsWith(".antivideo") && isGroup) {
            const args = text.split(" ")

            if (args[1] === "on") {
                const mode = args[2]

                if (["warn", "delete", "kick"].includes(mode)) {
                    antiVideoSettings[from] = mode
                    await sock.sendMessage(from, { text: `⚙️ Anti-video enabled: *${mode.toUpperCase()}*` })
                }
            }

            if (args[1] === "off") {
                delete antiVideoSettings[from]
                await sock.sendMessage(from, { text: "❌ Anti-video disabled" })
            }
        }

        // ================= VIDEO DETECTION =================
        if (isGroup && antiVideoSettings[from]) {
            if (msg.message.videoMessage) {
                const mode = antiVideoSettings[from]

                if (mode === "warn") {
                    await sock.sendMessage(from, { text: `⚠️ @${sender.split("@")[0]} sending videos is not allowed!`, mentions: [sender] })
                }

                if (mode === "delete") {
                    await sock.sendMessage(from, { delete: msg.key })
                }

                if (mode === "kick") {
                    await sock.groupParticipantsUpdate(from, [sender], "remove")
                }
            }
        }
    })
}

startBot()

