        // VERIFICATION (Admin & User only)
        if (role === 'admin' || role === 'user') {
            if (!verification_code) {
                const code = Math.floor(100000 + Math.random() * 900000).toString();
                const expires = new Date(Date.now() + 15 * 60 * 1000);

                await connection.query('DELETE FROM email_verification_codes WHERE user_id = ?', [user.user_id]);
                await connection.query('INSERT INTO email_verification_codes (user_id, code, expires_at) VALUES (?, ?, ?)', [user.user_id, code, expires]);

                const transporter = createTransporter();
                await transporter.sendMail({
                    from: `"H20ps Water" <${process.env.GMAIL_USER}>`,
                    to: user.email,
                    subject: 'Your Login Code',
                    html: `<h2>Your code: ${code}</h2>`
                });

                return res.render(template, {
                    error: 'Code sent to your email!',
                    email: user.email,
                    password: password,
                    showVerificationInput: true
                });
            }

            // Verify the code
            const [codes] = await connection.query(
                'SELECT * FROM email_verification_codes WHERE user_id = ? AND code = ? AND expires_at > NOW()',
                [user.user_id, verification_code]
            );

            if (codes.length === 0) {
                req.session.loginAttempts++;
                req.session.lastAttemptTime = Date.now();
                return res.render(template, { error: 'Invalid or expired code', email: user.email, password: password, showVerificationInput: true });
            }

            await connection.query('DELETE FROM email_verification_codes WHERE user_id = ?', [user.user_id]);
        }

        // LOGIN SUCCESS
        req.session.loginAttempts = 0;
        req.session.user = { id: user.user_id, username: user.username, role: user.role, email: user.email };

        // Precise Redirects
        if (user.role === 'admin') return res.redirect('/admin');
        if (user.role === 'staff') return res.redirect('/staff');
        return res.redirect('/');
